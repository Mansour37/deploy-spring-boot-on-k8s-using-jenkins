cat > Jenkinsfile <<'EOF'
pipeline {
  agent any

  tools { maven 'Maven' }

  environment {
    dockerimagename = "mansour38/spring-boot-k8s"
    // SONAR_AUTH_TOKEN doit exister dans Jenkins (Secret Text) si besoin.
  }

  stages {

    stage('Compilation') {
      steps {
        echo '⚙️ Compilation du projet Spring Boot...'
        sh 'mvn clean package -DskipTests'
      }
    }

    stage('Tests') {
      steps {
        echo '🧪 Exécution des tests unitaires Maven...'
        sh 'mvn test'
      }
    }

    stage('SAST - SonarQube') {
      steps {
        echo '🔒 Analyse SAST SonarQube...'
        withSonarQubeEnv('SonarQube') {
          sh 'mvn sonar:sonar -Dsonar.projectKey=spring-boot-k8s -Dsonar.host.url=http://localhost:9000 -Dsonar.login=$SONAR_AUTH_TOKEN'
        }
      }
    }

    stage('SCA - Trivy (repo)') {
      steps {
        sh '''
          set -e
          export PATH="/usr/local/bin:$PATH"
          mkdir -p reports

          trivy fs . \
            --security-checks vuln,secret,config \
            --severity HIGH,CRITICAL \
            --exit-code 1 \
            --ignore-unfixed \
            -f json -o reports/trivy-fs.json

          if [ -f /usr/local/share/trivy-html.tpl ]; then
            trivy fs . \
              --security-checks vuln,secret,config \
              --severity HIGH,CRITICAL \
              --ignore-unfixed \
              --format template \
              --template "@/usr/local/share/trivy-html.tpl" \
              -o reports/trivy-fs.html || true
          fi
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        echo '🐳 Construction de l’image Docker...'
        script { dockerImage = docker.build("${dockerimagename}:${BUILD_NUMBER}", ".") }
      }
    }

    stage('Trivy - Image Docker') {
      steps {
        script {
          def tag = "${dockerimagename}:${BUILD_NUMBER}"
          sh """
            set -e
            export PATH=\"/usr/local/bin:\$PATH\"
            mkdir -p reports

            trivy image ${tag} \
              --severity HIGH,CRITICAL \
              --exit-code 1 \
              --ignore-unfixed \
              -f json -o reports/trivy-image.json

            if [ -f /usr/local/share/trivy-html.tpl ]; then
              trivy image ${tag} \
                --severity HIGH,CRITICAL \
                --ignore-unfixed \
                --format template \
                --template "@/usr/local/share/trivy-html.tpl" \
                -o reports/trivy-image.html || true
            fi
          """
        }
      }
    }

    stage('Pushing Image') {
      environment { registryCredential = 'dockerhub-credentials' }
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', registryCredential) {
            dockerImage.push("latest")
            dockerImage.push("${BUILD_NUMBER}")
          }
        }
      }
    }

    stage('Déploiement') {
      steps {
        echo '📦 Déploiement sur Kubernetes...'
        withKubeConfig([credentialsId: 'mykubeconfig', serverUrl: 'https://192.168.49.2:8443']) {
          sh 'kubectl apply -f deployment-k8s.yaml'
          sh 'kubectl get pods -o wide'
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'reports/*.json, reports/*.html', fingerprint: true, allowEmptyArchive: true
    }
    success {
      echo '✅ OK : Compile → Test → SAST → SCA → Build → Image Scan → Push → Deploy.'
    }
    failure {
      echo '❌ KO — vérifie les logs (Trivy/Sonar peuvent échouer sur HIGH/CRITICAL).'
    }
  }
}
EOF
