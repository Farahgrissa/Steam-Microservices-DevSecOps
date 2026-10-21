pipeline {
    agent any

    options {
        timeout(time: 120, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    parameters {
        string(name: 'IMAGE_TAG',
               defaultValue: '1.0.0',
               description: 'Version des images a construire et deployer')
        booleanParam(name: 'RUN_TESTS',
                     defaultValue: true,
                     description: 'Executer les tests unitaires ?')
        booleanParam(name: 'DEPLOY',
                     defaultValue: true,
                     description: 'Deployer apres le build ?')
        booleanParam(name: 'SECURITY_GATES',
                     defaultValue: true,
                     description: 'Barrieres de securite bloquantes ? (decoche = mode observation)')
    }

    environment {
        HARBOR_REGISTRY = 'localhost:443'
        HARBOR_PROJECT  = 'library'
        COMPOSE_FILE    = 'docker-compose.deploy.yml'
        JENKINS_CONTAINER = 'jenkins'
        TRIVY_IMAGE       = 'aquasec/trivy:latest'
        TRIVY_CACHE       = '/opt/trivy-cache'
        ODC_IMAGE         = 'owasp/dependency-check:13.0.0'
        ODC_DATA          = '/opt/odc-data'
        ZAP_IMAGE         = 'zaproxy/zap-stable'
        ZAP_WRK           = '/opt/zap-wrk'
    }

    stages {

        stage('Build & Push') {
            steps {
                script {
                    def services = ['config-server', 'discovery-service', 'gateway', 'user-service']

                    services.each { svc ->
                        echo "==================== ${svc} ===================="

                        def imageName = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${svc}:${params.IMAGE_TAG}"
                        docker.image('maven:3.9-eclipse-temurin-17').inside('-v maven-cache:/root/.m2') {
                            dir(svc) {
                                if (params.RUN_TESTS) {
                                    sh 'mvn -Dmaven.wagon.http.retryHandler.count=5 clean package'
                                } else {
                                    sh 'mvn -Dmaven.wagon.http.retryHandler.count=5 clean package -DskipTests'
                                }
                            }
                        }
                        junit testResults: "${svc}/target/surefire-reports/*.xml",
                              allowEmptyResults: true

                        // SCA dependances : OWASP Dependency-Check
                        sh """
                            docker run --rm \
                              --volumes-from ${JENKINS_CONTAINER} \
                              -v ${ODC_DATA}:/usr/share/dependency-check/data \
                              ${ODC_IMAGE} \
                              --scan "${env.WORKSPACE}/${svc}/target" \
                              --project "${svc}" \
                              --format HTML --format JSON \
                              --out "${env.WORKSPACE}/${svc}/odc-report" \
                              --noupdate \
                              --disableCentral \
                              --disableOssIndex \
                              --disableRetireJs \
                              --disableNodeAudit \
                              --disableYarnAudit
                        """

                        // SAST + gate
                        dir(svc) {
                            withSonarQubeEnv('sonar') {
                                docker.image('maven:3.9-eclipse-temurin-17').inside('-v maven-cache:/root/.m2 --network gitea_gitea') {
                                    sh "mvn -Dmaven.wagon.http.retryHandler.count=5 org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar -Dsonar.projectKey=${svc}"
                                }
                            }
                        }
                        timeout(time: 5, unit: 'MINUTES') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                if (params.SECURITY_GATES) {
                                    error "SAST : Quality Gate echoue pour ${svc} : ${qg.status}"
                                } else {
                                    unstable("SAST : Quality Gate ${qg.status} pour ${svc} (mode observation)")
                                }
                            }
                        }
                        dir(svc) {
                            sh "docker build -f Dockerfile.ci -t ${imageName} ."
                        }
                        sh """
                            docker run --rm \
                              -v /var/run/docker.sock:/var/run/docker.sock \
                              -v ${TRIVY_CACHE}:/root/.cache/trivy \
                              ${TRIVY_IMAGE} image \
                              --exit-code 0 \
                              --skip-db-update \
                              --skip-java-db-update \
                              --scanners vuln \
                              --severity HIGH,CRITICAL \
                              --no-progress \
                              --timeout 20m \
                              ${imageName}
                        """

                        // SCA image : gate (CRITICAL corrigeables)
                        // Mode blocage : echec, l'image n'est pas publiee.
                        // Mode observation : build "unstable", la chaine continue.
                        def scaStatus = sh(returnStatus: true, script: """
                            docker run --rm \
                              -v /var/run/docker.sock:/var/run/docker.sock \
                              -v ${TRIVY_CACHE}:/root/.cache/trivy \
                              ${TRIVY_IMAGE} image \
                              --exit-code 1 \
                              --skip-db-update \
                              --skip-java-db-update \
                              --scanners vuln \
                              --severity CRITICAL \
                              --ignore-unfixed \
                              --no-progress \
                              --timeout 20m \
                              ${imageName}
                        """)
                        if (scaStatus != 0) {
                            if (params.SECURITY_GATES) {
                                error "SCA : vulnerabilites CRITICAL corrigeables dans ${svc} - publication bloquee"
                            } else {
                                unstable("SCA : vulnerabilites CRITICAL dans ${svc} (mode observation)")
                            }
                        }

                        // Push
                        retry(2) {
                            withCredentials([usernamePassword(
                                credentialsId: 'harbor-robot',
                                usernameVariable: 'HARBOR_USR',
                                passwordVariable: 'HARBOR_PSW'
                            )]) {
                                sh """
                                    echo "\$HARBOR_PSW" | docker login ${HARBOR_REGISTRY} -u "\$HARBOR_USR" --password-stdin
                                    docker push ${imageName}
                                    docker logout ${HARBOR_REGISTRY}
                                """
                            }
                        }

                        sh "docker rmi ${imageName} || true"
                        echo "==================== ${svc} : OK ===================="
                    }
                }
            }
        }

                stage('Deploy') {
            when {
                expression { return params.DEPLOY }
            }
            steps {
                script {
                    def services = ['config-server', 'discovery-service', 'gateway', 'user-service']
                    services.each { svc ->
                        sh """
                            sed -i -E 's#(${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${svc}:)[^ ]+#\\1${params.IMAGE_TAG}#' ${COMPOSE_FILE}
                        """
                    }
                    echo "Compose mis a jour vers la version ${params.IMAGE_TAG}"

                    // Secrets : compte robot Harbor + compte root MongoDB (jamais dans Git)
                    withCredentials([
                        usernamePassword(credentialsId: 'harbor-robot',
                                         usernameVariable: 'HARBOR_USR',
                                         passwordVariable: 'HARBOR_PSW'),
                        usernamePassword(credentialsId: 'mongo-root',
                                         usernameVariable: 'MONGO_ROOT_USER',
                                         passwordVariable: 'MONGO_ROOT_PASSWORD')
                    ]) {
                        sh """
                            echo "\$HARBOR_PSW" | docker login ${HARBOR_REGISTRY} -u "\$HARBOR_USR" --password-stdin
                            docker compose -p steam -f ${COMPOSE_FILE} pull
                            docker compose -p steam -f ${COMPOSE_FILE} up -d --remove-orphans --wait --wait-timeout 300
                            docker logout ${HARBOR_REGISTRY}
                        """
                    }
                }
            }
        }
        stage('Health Check') {
            when {
                expression { return params.DEPLOY }
            }
            steps {
                sh """
                    docker compose -p steam -f ${COMPOSE_FILE} ps
                """
            }
        }

        stage('DAST - OWASP ZAP') {
            when {
                expression { return params.DEPLOY }
            }
            steps {
                sh """
                    docker rm -f zap-scan || true
                    docker run --name zap-scan \
                      --network steam_microservices \
                      -v ${ZAP_WRK}:/zap/wrk:rw \
                      ${ZAP_IMAGE} \
                      zap-baseline.py \
                      -t http://gateway:8222 \
                      -r zap-report.html \
                      -I
                    docker cp zap-scan:/zap/wrk/zap-report.html ./zap-report.html
                    docker rm -f zap-scan
                """
            }
            post {
                always {
                    sh 'docker rm -f zap-scan || true'
                    archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
                }
            }
        }

    }

    post {
        success {
            echo "SUCCES : build, push et deploiement termines (version ${params.IMAGE_TAG})."
        }
        unstable {
            echo "INSTABLE : failles detectees en mode observation, images publiees malgre tout (version ${params.IMAGE_TAG})."
        }
        failure {
            echo "ECHEC : la pipeline s'est arretee."
        }
        always {
            archiveArtifacts artifacts: '*/odc-report/*', allowEmptyArchive: true
            echo "Pipeline terminee a ${new Date()}"
        }
    }
}