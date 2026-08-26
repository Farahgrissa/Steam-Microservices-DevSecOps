pipeline {
    agent any

    options {
        timeout(time: 60, unit: 'MINUTES')
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
    }

    environment {
        HARBOR_REGISTRY = 'localhost:443'
        HARBOR_PROJECT  = 'library'
        SONAR_HOST      = 'http://192.168.91.129:9000'
        COMPOSE_FILE    = 'docker-compose.deploy.yml'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'http://gitea:3000/gitea/steam-microservices.git',
                    credentialsId: 'gitea-creds'
            }
        }

        // ── BUILD SEQUENTIEL
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
                                    sh 'mvn clean package'
                                } else {
                                    sh 'mvn clean package -DskipTests'
                                }
                            }
                        }
                        junit testResults: "${svc}/target/surefire-reports/*.xml",
                              allowEmptyResults: true

                        docker.image('maven:3.9-eclipse-temurin-17').inside('-v maven-cache:/root/.m2') {
                            dir(svc) {
                                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                                    sh """
                                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                                          -Dsonar.host.url=${SONAR_HOST} \
                                          -Dsonar.token=\$SONAR_TOKEN \
                                          -Dsonar.projectKey=${svc}
                                    """
                                }
                            }
                        }

                        dir(svc) {
                            sh "docker build -f Dockerfile.ci -t ${imageName} ."
                        }

                        sh """
                            docker run --rm \
                              -v /var/run/docker.sock:/var/run/docker.sock \
                              -v trivy-cache:/root/.cache/trivy \
                              aquasec/trivy:latest image \
                              --exit-code 0 \
                              --scanners vuln \
                              --severity HIGH,CRITICAL \
                              --no-progress \
                              --timeout 20m \
                              ${imageName}
                        """

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
                    withCredentials([usernamePassword(
                        credentialsId: 'harbor-robot',
                        usernameVariable: 'HARBOR_USR',
                        passwordVariable: 'HARBOR_PSW'
                    )]) {
                        sh """
                            echo "\$HARBOR_PSW" | docker login ${HARBOR_REGISTRY} -u "\$HARBOR_USR" --password-stdin
                            docker compose -f ${COMPOSE_FILE} pull
                            docker compose -f ${COMPOSE_FILE} up -d
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
                    echo "Attente du demarrage des services..."
                    sleep 30
                    docker compose -f ${COMPOSE_FILE} ps
                """
            }
        }
    }

    post {
        success {
            echo "SUCCÈS : build, push et deploiement termines (version ${params.IMAGE_TAG})."
        }
        failure {
            echo "ECHEC : la pipeline s'est arretee."
        }
        always {
            echo "Pipeline terminée à ${new Date()}"
        }
    }
}
