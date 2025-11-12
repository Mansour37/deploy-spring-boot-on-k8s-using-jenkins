// small update to test git push

pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        dockerimagename = "mansour38/spring-boot-k8s"
        registryCredential = 'dockerhub-credentials'
    }

    stages {

        // 1️⃣ Compilation du code
        stage('Compilation') {
            steps {
                echo '⚙️ Compilation du projet Spring Boot...'
                sh 'mvn clean package -DskipTests'
            }
        }

        // 2️⃣ Exécution des tests unitaires
        stage('Tests') {
            steps {
                echo '🧪 Exécution des tests unitaires Maven...'
                sh 'mvn test'
            }
        }

        // 3️⃣ Construction de l’image Docker
        stage('Build Docker Image') {
            steps {
                echo ' Construction de l’image Docker...'
                script {
                    dockerImage = docker.build("${dockerimagename}:${BUILD_NUMBER}", ".")
                }
            }
        }

        // 4️⃣ Push de l’image sur DockerHub
        // 4️⃣ Push de l’image sur DockerHub (Solution Alternative)
// 4️⃣ Push de l’image sur DockerHub (Solution Alternative)
    stage('Pushing Image') {
      environment {
               registryCredential = 'dockerhub-credentials'
           }
      steps{
        script {
          docker.withRegistry( 'https://registry.hub.docker.com', registryCredential ) {
            dockerImage.push("latest")
          }
        }
      }
    }


        // 5️⃣ Déploiement sur Kubernetes
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
