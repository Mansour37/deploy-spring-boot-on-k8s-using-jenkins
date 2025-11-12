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

        stage('SCA - Analyse des dépendances') {
          steps {
            echo '📦 Analyse des dépendances avec OWASP Dependency-Check...'
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
              sh '''
                mkdir -p reports
                if ! command -v dependency-check.sh >/dev/null 2>&1; then
                  echo "⚠️ OWASP Dependency-Check n'est pas installé sur ce serveur Jenkins."
                else
                  dependency-check.sh \
                    --project springboot-k8s \
                    --scan . \
                    --format HTML \
                    --out reports || true
                fi
              '''
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
        success {
            echo '✅ Pipeline DevOps exécuté avec succès (Compilation → Test → Build → Déploiement).'
        }
        failure {
            echo '❌ Le pipeline a échoué — vérifiez les logs Jenkins.'
        }
    }
}
