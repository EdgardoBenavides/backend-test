pipeline {
  agent any

  environment {
    IMAGE_TOOLING      = 'edgardobenavidesl/node-java-sonar-docker:latest'
    SONARQUBE_SERVER   = 'SonarQube'                     // nombre en Jenkins Global Config
    SONAR_PROJECT_KEY  = 'backend-test'

    // ⛳ IMPORTANTE: desde el clúster K8s "localhost" no sirve.
    // En Docker Desktop, el host es 'host.docker.internal'. Si tu Nexus corre en otro host, pon su IP/DNS:PUERTO.
    NEXUS_REGISTRY     = 'host.docker.internal:8082'

    IMAGE_NAME         = "${NEXUS_REGISTRY}/backend-test"
    BUILD_TAG          = "${env.BUILD_NUMBER}"
    MAX_IMAGES_TO_KEEP = '5'

    K8S_NAMESPACE      = 'default'
    DEPLOYMENT_FILE    = 'kubernetes.yaml'
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
              # Normaliza rutas de lcov para Sonar (tolerante si no existe)
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
            withCredentials([usernamePassword(
              credentialsId: 'nexus-credentials',   // credencial tipo "Username with password"
              usernameVariable: 'NEXUS_USER',
              passwordVariable: 'NEXUS_PASS'
            )]) {
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
        file(credentialsId: 'kubeconfig',       variable: 'KUBECONFIG_FILE'),
        usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')
      ]) {
        sh '''
          set -eux
          NS="${K8S_NAMESPACE}"
          DF="${DEPLOYMENT_FILE:-kubernetes.yaml}"
          [ -f "$DF" ] || { echo "No se encontró $DF"; ls -la; exit 2; }

          # 1) Preparar ~/.kube/config como DIRECTORIO (no archivo suelto)
          #    - Creamos .kube/ en el workspace y copiamos ahí el kubeconfig como "config"
          mkdir -p "$PWD/.kube"
          SRC="$KUBECONFIG_FILE"
          if [ -f "$SRC" ]; then
            cp "$SRC" "$PWD/.kube/config"
          elif [ -d "$SRC" ] && [ -f "$SRC/config" ]; then
            cp "$SRC/config" "$PWD/.kube/config"
          else
            echo "Credencial kubeconfig inválida (no es archivo ni directorio con 'config')"; exit 2
          fi
          chmod 600 "$PWD/.kube/config"
          awk '/server:/ {print "Kubeconfig server:", $2; exit}' "$PWD/.kube/config" || true

          # Montamos la CARPETA .kube al /root/.kube de la imagen
          KMOUNT="-v $PWD/.kube:/root/.kube:ro"

          # 2) (debug rápido) Asegurar que kubectl puede LEER el kubeconfig
          docker run --rm $KMOUNT bitnami/kubectl:latest config view --minify >/dev/null

          # 3) Crear/actualizar imagePullSecret para Nexus
          docker run --rm $KMOUNT bitnami/kubectl:latest -n "$NS" create secret docker-registry nexus-docker \
            --docker-server=${NEXUS_REGISTRY} \
            --docker-username="$REG_USER" \
            --docker-password="$REG_PASS" \
            --docker-email="noreply@local" \
            --dry-run=client -o yaml | \
          docker run --rm -i $KMOUNT bitnami/kubectl:latest -n "$NS" apply -f -

          # 4) Aplicar manifiesto montando el workspace y usando -f "$DF"
          docker run --rm $KMOUNT -v "$PWD:/work" -w /work \
            bitnami/kubectl:latest -n "$NS" apply -f "$DF"

          # 5) Forzar imagen EXACTA del build
          docker run --rm $KMOUNT \
            bitnami/kubectl:latest -n "$NS" set image deployment/backend-test backend-test=${IMAGE_NAME}:${BUILD_NUMBER}

          # 6) Esperar rollout y verificar
          docker run --rm $KMOUNT \
            bitnami/kubectl:latest -n "$NS" rollout status deployment/backend-test --timeout=180s

          DR=$(docker run --rm $KMOUNT bitnami/kubectl:latest -n "$NS" get deploy backend-test -o jsonpath='{.spec.replicas}')
          AR=$(docker run --rm $KMOUNT bitnami/kubectl:latest -n "$NS" get deploy backend-test -o jsonpath='{.status.availableReplicas}')
          echo "Desired replicas: ${DR:-?} | Available replicas: ${AR:-0}"
          test -n "$DR" && [ "${AR:-0}" = "$DR" ]

          docker run --rm $KMOUNT \
            bitnami/kubectl:latest -n "$NS" get pods -l app=backend-test -o wide
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
