// Pipeline d'intégration continue de la bijouterie.
// Il construit les trois images Docker, leur donne deux tags et les envoie
// sur Docker Hub. Le déploiement est ensuite fait par Argo CD.
pipeline {
    // Le build peut tourner sur n'importe quel agent qui a Docker.
    agent any

    // Variables communes à tous les stages.
    environment {
        // Compte Docker Hub qui reçoit les images.
        REGISTRY = 'pamalick'
    }

    // Poll SCM : Jenkins regarde le dépôt toutes les 5 minutes environ
    // et lance un build s'il trouve un nouveau commit.
    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {

        // Checkout : récupère le code du dépôt Git configuré dans le job.
        // On garde aussi l'identifiant court du commit pour le 2e tag.
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }

        // Build : construit les trois images depuis leurs Dockerfiles.
        // Premier tag : le numéro du build Jenkins (jamais latest).
        stage('Build') {
            steps {
                sh 'docker build -t $REGISTRY/bijouterie-frontend:$BUILD_NUMBER ./frontend'
                sh 'docker build -t $REGISTRY/bijouterie-backend:$BUILD_NUMBER ./backend'
                sh 'docker build -t $REGISTRY/bijouterie-mysql:$BUILD_NUMBER ./database'
            }
        }

        // Tag : ajoute un deuxième tag, l'identifiant court du commit Git.
        // On peut ainsi retrouver le code exact qui a produit chaque image.
        stage('Tag') {
            steps {
                sh 'docker tag $REGISTRY/bijouterie-frontend:$BUILD_NUMBER $REGISTRY/bijouterie-frontend:$GIT_SHORT'
                sh 'docker tag $REGISTRY/bijouterie-backend:$BUILD_NUMBER $REGISTRY/bijouterie-backend:$GIT_SHORT'
                sh 'docker tag $REGISTRY/bijouterie-mysql:$BUILD_NUMBER $REGISTRY/bijouterie-mysql:$GIT_SHORT'
            }
        }

        // Push : se connecte à Docker Hub et envoie les deux tags de chaque image.
        // Le token vient du credential dockerhub-credentials : il n'est jamais
        // écrit dans ce fichier et Jenkins le masque dans la console.
        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                                  usernameVariable: 'DOCKERHUB_USER',
                                                  passwordVariable: 'DOCKERHUB_TOKEN')]) {
                    // --password-stdin : le token ne passe pas en argument de la commande.
                    sh 'echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin'
                }
                sh 'docker push $REGISTRY/bijouterie-frontend:$BUILD_NUMBER'
                sh 'docker push $REGISTRY/bijouterie-frontend:$GIT_SHORT'
                sh 'docker push $REGISTRY/bijouterie-backend:$BUILD_NUMBER'
                sh 'docker push $REGISTRY/bijouterie-backend:$GIT_SHORT'
                sh 'docker push $REGISTRY/bijouterie-mysql:$BUILD_NUMBER'
                sh 'docker push $REGISTRY/bijouterie-mysql:$GIT_SHORT'
            }
        }
    }

    // Après le build, qu'il réussisse ou non : on se déconnecte de Docker Hub.
    post {
        always {
            sh 'docker logout || true'
        }
    }
}
