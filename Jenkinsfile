pipeline {
    agent any

    environment {
        IMAGE_NAME = "edgardobenavidesl/backend-test"
        BUILD_TAG = "${new Date().format('yyyyMMddHHmmss')}"
        MAX_IMAGES_TO_KEEP = 5
        SONAR_PROJECT_KEY = "backend-test"
        NEXUS_URL = "nexus_repo:8082"
        KUBE_CONFIG = "/home/jenkins/.kube/config"
        DEPLOYMENT_FILE = "kubernetes.yaml"
        SONAR_HOST_URL = "http://sonarqube:9000"
        SONAR_AUTH_TOKEN = credentials('sonarqube-cred')
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        stage('Instalación de dependencias y tests') {
            agent {
                docker {
                    image 'cimg/node:22.2.0'
                    args '--network devnet'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -eux
                    npm ci
                    npm run test:cov

                    if [ ! -f coverage/lcov.info ]; then
                        echo "ERROR: No se generó coverage/lcov.info"
                        exit 1
                    fi

                    echo "Normalizando rutas en lcov.info..."
                    sed -i 's|SF:.*/src|SF:src|g' coverage/lcov.info
                    sed -i 's|\\\\|/|g' coverage/lcov.info
                '''
            }
        }

        stage('Build aplicación') {
            agent {
                docker {
                    image 'cimg/node:22.2.0'
                    args '--network devnet'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -eux
                    npm run build
                '''
            }
        }

        stage('Debug SonarQube desde contenedor') {
            agent {
                docker {
                    image 'cimg/node:22.2.0'
                    args '--network devnet'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -eux
                    echo "Probando conexión a SonarQube desde contenedor..."
                    curl -v -u ${SONAR_AUTH_TOKEN}: ${SONAR_HOST_URL}/api/system/health || echo "No se pudo conectar desde contenedor"
                '''
            }
        }

        stage('Análisis SonarQube') {
            agent {
                docker {
                    image 'cimg/node:22.2.0'
                    args '--network devnet'
                    reuseNode true
                }
            }
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            set -eux
                            npx sonarqube-scanner \
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                -Dsonar.sources=src \
                                -Dsonar.tests=src \
                                -Dsonar.test.inclusions=**/*.spec.ts \
                                -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                                -Dsonar.exclusions=node_modules/**,dist/** \
                                -Dsonar.coverage.exclusions=**/*.spec.ts \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.token=${SONAR_AUTH_TOKEN}
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    echo "Esperando 15s para que SonarQube procese el análisis..."
                    sleep 15
                    timeout(time: 10, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline detenido por Quality Gate: ${qg.status}"
                        } else {
                            echo "Quality Gate OK"
                        }
                    }
                }
            }
        }

   stage('Docker Build & Push') {
    steps {
        script {
            // Variables
            def nexusHost = "172.20.0.4:8082" // IP de nexus_repo
            def imageTag = "${IMAGE_NAME}:${BUILD_TAG}"

            withCredentials([
                usernamePassword(credentialsId: 'NEXUS_CREDENTIALS_ID', 
                                 passwordVariable: 'NEXUS_PASSWORD', 
                                 usernameVariable: 'NEXUS_USER')
            ]) {
                sh """
                    set -eux

                    echo 'Eliminando imágenes antiguas...'
                    docker images ${IMAGE_NAME} --format '{{.Repository}}:{{.Tag}}' | tail -n +6 | sort -r | xargs -r docker rmi -f || true

                    echo 'Login en Nexus...'
                    echo "$NEXUS_PASSWORD" | docker login http://${nexusHost} -u "$NEXUS_USER" --password-stdin

                    echo 'Construyendo imagen Docker...'
                    docker build -t ${imageTag} .

                    echo 'Tagueando imagen para Nexus...'
                    docker tag ${imageTag} ${nexusHost}/${IMAGE_NAME}:${BUILD_TAG}

                    echo 'Pusheando imagen a Nexus...'
                    docker push ${nexusHost}/${IMAGE_NAME}:${BUILD_TAG}
                """
            }
        }
    }
}






        stage('Verificar Nexus') {
            agent any
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-credentials', 
                                                     usernameVariable: 'NEXUS_USER', 
                                                     passwordVariable: 'NEXUS_PASSWORD')]) {
                        sh '''
                            set -eux
                            echo "Verificando que la imagen Docker exista en Nexus..."
                            RESPONSE=$(curl -s -u $NEXUS_USER:$NEXUS_PASSWORD \
                                "http://nexus_repo:8081/service/rest/v1/components?repository=dockerreponexus" \
                                | grep "$IMAGE_NAME" | grep "$BUILD_TAG" || true)

                            if [ -z "$RESPONSE" ]; then
                                echo "No se encontró la imagen $IMAGE_NAME:$BUILD_TAG en Nexus"
                                exit 1
                            else
                                echo "Imagen $IMAGE_NAME:$BUILD_TAG encontrada en Nexus"
                            fi
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finalizado'
        }
    }
}
