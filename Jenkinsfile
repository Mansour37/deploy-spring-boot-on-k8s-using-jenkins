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

        stage('Build Docker Image') {
            steps {
                echo '🐳 Construction de l’image Docker...'
                script {
                    dockerImage = docker.build("${dockerimagename}:${BUILD_NUMBER}", ".")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        // Connexion Docker
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

                        // Push avec deux tags
                        sh "docker push ${dockerimagename}:${BUILD_NUMBER}"
                        sh "docker tag ${dockerimagename}:${BUILD_NUMBER} ${dockerimagename}:latest"
                        sh "docker push ${dockerimagename}:latest"

                        sh 'docker logout'
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
