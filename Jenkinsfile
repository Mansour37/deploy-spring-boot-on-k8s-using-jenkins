pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        dockerimagename = "mansour38/spring-boot-k8s"
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

        stage('SAST - Analyse de sécurité du code') {
            steps {
                echo '🔒 Analyse de sécurité SAST avec SonarQube...'
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=spring-boot-k8s -Dsonar.host.url=http://localhost:9000 -Dsonar.login=$SONAR_AUTH_TOKEN'
                }
            }
        }

        stage('Secrets Scan - Gitleaks') {
            steps {
                echo '🔑 Scan des secrets avec Gitleaks...'
                sh '''
                    mkdir -p reports
                    gitleaks detect \
                        --source . \
                        --no-git \
                        --report-path reports/gitleaks.json \
                        --report-format json \
                        --exit-code 1 || true
                '''
            }
        }


         stage('SCA - Trivy (repo)') {
            steps {
              script {
                def status = sh(
                  script: '''
                    export PATH="/usr/local/bin:$PATH"
                    mkdir -p reports
                    trivy fs . \
                      --scanners vuln,misconfig,secret \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --ignore-unfixed \
                      -f json -o reports/trivy-fs.json || true
                  ''',
                  returnStatus: true
                )

                if (status == 1) {
                  currentBuild.result = 'UNSTABLE'
                  echo '⚠️ Vulnérabilités détectées (HIGH/CRITICAL) — voir le rapport Trivy.'
                }
              }
            }
          }





        stage('Build Docker Image') {
            steps {
                echo '🐳 Construction de l’image Docker...'
                script {
                    dockerImage = docker.build("${dockerimagename}:${BUILD_NUMBER}", ".")
                }
            }
        }

        stage('Trivy (Scan Docker image)') {
            steps {
                script {
                    def status = sh(
                        script: """
                            export PATH="/usr/local/bin:$PATH"
                            mkdir -p reports

                            trivy image ${dockerimagename}:${BUILD_NUMBER} \
                              --severity HIGH,CRITICAL \
                              --exit-code 1 \
                              --ignore-unfixed \
                              -f json -o reports/trivy-image.json || true

                            if [ -f /usr/local/share/trivy-html.tpl ]; then
                              trivy image ${dockerimagename}:${BUILD_NUMBER} \
                                --severity HIGH,CRITICAL \
                                --ignore-unfixed \
                                --format template \
                                --template "@/usr/local/share/trivy-html.tpl" \
                                -o reports/trivy-image.html || true
                            fi
                        """,
                        returnStatus: true
                    )

                    if (status == 1) {
                        currentBuild.result = 'UNSTABLE'
                        echo '⚠️ Vulnérabilités détectées dans l’image Docker.'
                    }
                }
            }
        }
        stage('Pushing Image') {
            environment {
                registryCredential = 'dockerhub-credentials'
            }
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
                echo '📦 Déploiement sur le cluster Kubernetes...'
                withKubeConfig([credentialsId: 'mykubeconfig', serverUrl: 'https://192.168.49.2:8443']) {
                    sh 'kubectl apply -f deployment-k8s.yaml'
                    sh 'kubectl get pods -o wide'
                }
            }
        }
    }

        post {
      always {
        archiveArtifacts artifacts: 'reports/*.json, reports/*.html',
          fingerprint: true, allowEmptyArchive: true
      }
    }
}
