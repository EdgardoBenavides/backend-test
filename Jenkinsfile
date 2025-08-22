pipeline {
    agent {
        docker {
            image 'edgardobenavidesl/backend-test'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
            reuseNode true
        }
    }

    environment {
        IMAGE_NAME = "edgardobenavidesl/backend-test"
    }

    stages {
        stage('Instalación de dependencias') {
            steps { sh 'npm install' }
        }

        stage('Pruebas automatizadas') {
            steps { sh 'npm run test:cov' }
        }

        stage('Construcción de aplicación') {
            steps { sh 'npm run build' }
        }

        stage('Quality Assurance') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    script {
                        docker.image('sonarsource/sonar-scanner-cli:latest').inside('--network dockercompose_devnet') {
                            sh '''
                                sonar-scanner \
                                -Dsonar.projectKey=backend-test \
                                -Dsonar.sources=src \
                                -Dsonar.tests=src \
                                -Dsonar.test.inclusions=src/**/*.spec.ts \
                                -Dsonar.login=$SONAR_AUTH_TOKEN
                            '''
                        }
                    }
                }
            }
        }

        stage('Quality Gate'){
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Empaquetado y push Docker') {
            steps {
                script {
                    def app = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")

                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        app.push()
                        app.push("ebl")
                    }

                    docker.withRegistry('http://localhost:8082', 'nexus-credentials') {
                        sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} localhost:8082/${IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} localhost:8082/${IMAGE_NAME}:latest"

                        sh "docker push localhost:8082/${IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker push localhost:8082/${IMAGE_NAME}:latest"
                    }
                }
            }
        }
    }
}

 


// pipeline {
//     agent {
//         docker {
//             image 'node:22'
//             args '-v /var/run/docker.sock:/var/run/docker.sock'
//             reuseNode true
//         }
//     }

//     environment {
//         IMAGE_NAME = "edgardobenavidesl/backend-test"
//         BUILD_TAG = "${new Date().format('yyyyMMddHHmmss')}"
//         MAX_IMAGES_TO_KEEP = 5
//         SONAR_HOST_URL = "http://host.docker.internal:9000"
//     }

//     stages {
//         stage('Checkout SCM') {
//             steps {
//                 checkout([$class: 'GitSCM',
//                     branches: [[name: 'dev']],
//                     doGenerateSubmoduleConfigurations: false,
//                     extensions: [],
//                     userRemoteConfigs: [[
//                         url: 'https://github.com/EdgardoBenavides/backend-test.git',
//                         credentialsId: 'Githubpas'
//                     ]]
//                 ])
//             }
//         }

//         stage('Instalación de dependencias') {
//             steps {
//                 sh 'npm ci'
//             }
//         }

//         stage('Ejecución de pruebas automatizadas') {
//             steps {
//                 sh 'npm run test:cov'
//             }
//         }

//         stage('Construcción de aplicación') {
//             steps {
//                 sh 'npm run build'
//             }
//         }

//         stage('Quality Assurance') {
//             steps {
//                 withSonarQubeEnv('SonarQube') {
//                     sh '''
//                     npm install -g sonar-scanner
//                     sonar-scanner \
//                         -Dsonar.projectKey=backend-test \
//                         -Dsonar.sources=src \
//                         -Dsonar.tests=src \
//                         -Dsonar.test.inclusions=src/**/*.spec.ts \
//                         -Dsonar.exclusions=coverage/**,src/config/configuration.ts,src/**/*.spec.ts \
//                         -Dsonar.coverage.exclusions=src/**/*.spec.ts \
//                         -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
//                         -Dsonar.login=$SONAR_AUTH_TOKEN
//                     '''
//                 }
//             }
//         }

//         stage('Quality Gate') {
//             steps {
//                 timeout(time: 10, unit: 'MINUTES') { 
//                     script {
//                         def gate = waitForQualityGate()
//                         if (gate.status != 'OK') {
//                             error "Quality Gate failed with status: ${gate.status}"
//                         } else {
//                             echo "Quality Gate passed!"
//                         }
//                     }
//                 }
//             }
//         }

//         stage('Empaquetado y push Docker') {
//             steps {
//                 script {
//                     // Limpiar imágenes antiguas
//                     sh """
//                         docker images ${IMAGE_NAME} --format "{{.Repository}}:{{.Tag}}" \
//                         | sort -r | tail -n +\$((MAX_IMAGES_TO_KEEP + 1)) | xargs -r docker rmi -f || true
//                     """

//                     // Docker Hub
//                     docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
//                         def app = docker.build("${IMAGE_NAME}:${BUILD_TAG}")

//                         sh "docker rmi ${IMAGE_NAME}:ebl || true"
//                         sh "docker tag ${IMAGE_NAME}:${BUILD_TAG} ${IMAGE_NAME}:ebl"

//                         app.push("${BUILD_TAG}")
//                         sh "docker push ${IMAGE_NAME}:ebl"
//                     }

//                     // Nexus
//                     def nexusHost = sh(
//                         script: 'ping -c 1 nexus >/dev/null 2>&1 && echo "nexus" || echo "localhost"',
//                         returnStdout: true
//                     ).trim()

//                     echo "Usando Nexus host: ${nexusHost}"

//                     docker.withRegistry("http://${nexusHost}:8082", 'nexus-credentials') {
//                         sh "docker tag ${IMAGE_NAME}:${BUILD_TAG} ${nexusHost}:8082/${IMAGE_NAME}:${BUILD_TAG}"
//                         sh "docker push ${nexusHost}:8082/${IMAGE_NAME}:${BUILD_TAG}"

//                         sh "docker tag ${IMAGE_NAME}:${BUILD_TAG} ${nexusHost}:8082/${IMAGE_NAME}:ebl"
//                         sh "docker push ${nexusHost}:8082/${IMAGE_NAME}:ebl"
//                     }
//                 }
//             }
//         }
//     }

//     post {
//         always {
//             echo 'Pipeline finalizado'
//         }
//     }
// }
