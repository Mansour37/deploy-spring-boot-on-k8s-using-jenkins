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
    stage('Push Docker Image') {
        steps {
            script {
                withCredentials([usernamePassword(
                    credentialsId: env.registryCredential, 
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) 
                {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

                    def imageBuildName = "mansour38/spring-boot-k8s"
                    def imageTag = "${imageBuildName}:${env.BUILD_NUMBER}"
                    
                    // Correction du tag manquant :
                    // La Stage 3 construit l'image, MAIS la commande "docker build" taggue l'image
                    // avec le nom donné par le plugin Docker. Si l'image n'est pas taguée correctement, 
                    // les push suivants échouent.
                    
                    // Ajout d'une commande pour s'assurer que l'image est bien taguée
                    sh "docker tag ${imageBuildName}:latest ${imageTag}" // Tag l'image construite avec le BUILD_NUMBER
                    sh "docker tag ${imageBuildName}:latest ${imageBuildName}:latest" // S'assure que le latest est correct
                    
                    sh "docker push ${imageTag}"
                    sh "docker push ${imageBuildName}:latest"
                    
                    sh 'docker logout'
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
