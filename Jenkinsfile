pipeline {
  agent any

  environment {
    IMAGE_TOOLING      = 'edgardobenavidesl/node-java-sonar-docker:latest'
    SONARQUBE_SERVER   = 'SonarQube'                     // nombre en Jenkins Global Config
    SONAR_PROJECT_KEY  = 'backend-test'
    NEXUS_REGISTRY     = 'localhost:8082'                // ⚠️ usa host/IP alcanzable por los nodos K8s (no "localhost")
    IMAGE_NAME         = "${NEXUS_REGISTRY}/backend-test"
    BUILD_TAG          = "${env.BUILD_NUMBER}"
    MAX_IMAGES_TO_KEEP = '5'
    K8S_NAMESPACE      = 'default'
  }

  options { timeout(time: 45, unit: 'MINUTES') }

  stages {
    stage('Checkout SCM') {
      steps { checkout scm }
    }

    stage('Install dependencies') {
      steps {
        script {
          docker.image(env.IMAGE_TOOLING).inside('-v /var/run/docker.sock:/var/run/docker.sock --network devnet') {
            sh '''
              set -eux
              npm ci
            '''
          }
        }
      }
    }

    stage('Run tests & coverage') {
      steps {
        script {
          docker.image(env.IMAGE_TOOLING).inside('-v /var/run/docker.sock:/var/run/docker.sock --network devnet') {
            sh '''
              set -eux
              npm run test:cov
              # Normaliza rutas de lcov para Sonar
              sed -i 's|SF:.*/src|SF:src|g' coverage/lcov.info || true
              sed -i 's#\\\\#/#g' coverage/lcov.info || true
            '''
          }
        }
      }
    }

    stage('Build app') {
      steps {
        script {
          docker.image(env.IMAGE_TOOLING).inside('-v /var/run/docker.sock:/var/run/docker.sock --network devnet') {
            sh '''
              set -eux
              npm run build
            '''
          }
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        script {
          docker.image(env.IMAGE_TOOLING).inside('-v /var/run/docker.sock:/var/run/docker.sock --network devnet') {
            withSonarQubeEnv(SONARQUBE_SERVER) {
              withCredentials([string(credentialsId: 'sonarqube-cred', variable: 'SONAR_TOKEN')]) {
                sh '''
                  set -eux
                  PK="${SONAR_PROJECT_KEY:-backend-test}"
                  [ -n "$PK" ] || { echo "ERROR: SONAR_PROJECT_KEY vacío"; exit 2; }

                  sonar-scanner \
                    -Dsonar.projectKey=$PK \
                    -Dsonar.projectName=$PK \
                    -Dsonar.sources=src \
                    -Dsonar.tests=src \
                    -Dsonar.test.inclusions=**/*.spec.ts \
                    -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                    -Dsonar.exclusions=node_modules/**,dist/** \
                    -Dsonar.coverage.exclusions=**/*.spec.ts \
                    -Dsonar.host.url=http://sonarqube:9000 \
                    -Dsonar.login="$SONAR_TOKEN"
                '''
              }
            }
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          script {
            def qg = waitForQualityGate abortPipeline: true
            echo "Quality Gate: ${qg.status}"
          }
        }
      }
    }

    stage('Docker Build & Push (Nexus)') {
      steps {
        script {
          docker.image(env.IMAGE_TOOLING).inside('-v /var/run/docker.sock:/var/run/docker.sock --network devnet') {
            // Asegúrate que 'nexus-credentials' sea de tipo "Username with password"
            withCredentials([usernamePassword(credentialsId: 'nexus-credentials',
              usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
              sh '''
                set -eux
                echo "$NEXUS_PASS" | docker login -u "$NEXUS_USER" --password-stdin http://${NEXUS_REGISTRY}

                docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .
                docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                docker push ${IMAGE_NAME}:latest

                docker image prune -f || true
              '''
            }
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        script {
          withCredentials([
            file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')
          ]) {
            sh '''
              set -eux
              NS="${K8S_NAMESPACE}"
              KCONF="$KUBECONFIG_FILE"

              # Aplica manifiesto (monta workspace)
              docker run --rm --network devnet \
                -v "$KCONF:/root/.kube/config:ro" \
                -v "$PWD:/work" -w /work \
                bitnami/kubectl:latest \
                -n "$NS" apply -f kubernetes.yaml

              # Fuerza actualización a la última imagen
              docker run --rm --network devnet \
                -v "$KCONF:/root/.kube/config:ro" \
                bitnami/kubectl:latest \
                -n "$NS" set image deployment/backend-test backend-test=${IMAGE_NAME}:latest

              # Espera rollout
              docker run --rm --network devnet \
                -v "$KCONF:/root/.kube/config:ro" \
                bitnami/kubectl:latest \
                -n "$NS" rollout status deployment/backend-test --timeout=180s

              # Verificación: disponibles == deseadas
              DR=$(docker run --rm --network devnet \
                  -v "$KCONF:/root/.kube/config:ro" \
                  bitnami/kubectl:latest \
                  -n "$NS" get deploy backend-test -o jsonpath='{.spec.replicas}')
              AR=$(docker run --rm --network devnet \
                  -v "$KCONF:/root/.kube/config:ro" \
                  bitnami/kubectl:latest \
                  -n "$NS" get deploy backend-test -o jsonpath='{.status.availableReplicas}')
              echo "Desired replicas: ${DR:-?} | Available replicas: ${AR:-0}"
              test -n "$DR" && [ "${AR:-0}" = "$DR" ]

              docker run --rm --network devnet \
                -v "$KCONF:/root/.kube/config:ro" \
                bitnami/kubectl:latest \
                -n "$NS" get pods -l app=backend-test -o wide
            '''
          }
        }
      }
    }

  }

  post {
    always {
      echo "Pipeline finalizado."
      deleteDir()
    }
  }
}
