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
        // Dépôt GitHub où le stage Update manifests pousse ses commits (sans https://).
        GIT_REPO = 'github.com/pa-malick/bijouterie-microservices.git'
    }

    // Poll SCM : Jenkins regarde le dépôt toutes les 5 minutes environ
    // et lance un build s'il trouve un nouveau commit.
    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {

        // Checkout : récupère le code du dépôt Git configuré dans le job.
        // On garde aussi l'identifiant court du commit pour le 2e tag.
        // Si le dernier commit vient du stage Update manifests ([skip ci]),
        // on ne relance pas tout : sinon chaque build en déclencherait un autre.
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.SKIP_CI = sh(script: 'git log -1 --pretty=%B', returnStdout: true).contains('[skip ci]').toString()
                    if (env.SKIP_CI == 'true') {
                        currentBuild.description = 'Commit de mise à jour des tags : rien à construire'
                    }
                }
            }
        }

        // Build : construit les trois images depuis leurs Dockerfiles.
        // Premier tag : le numéro du build Jenkins (jamais latest).
        stage('Build') {
            when { expression { env.SKIP_CI != 'true' } }
            steps {
                sh 'docker build -t $REGISTRY/bijouterie-frontend:$BUILD_NUMBER ./frontend'
                sh 'docker build -t $REGISTRY/bijouterie-backend:$BUILD_NUMBER ./backend'
                sh 'docker build -t $REGISTRY/bijouterie-mysql:$BUILD_NUMBER ./database'
            }
        }

        // Tag : ajoute un deuxième tag, l'identifiant court du commit Git.
        // On peut ainsi retrouver le code exact qui a produit chaque image.
        stage('Tag') {
            when { expression { env.SKIP_CI != 'true' } }
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
            when { expression { env.SKIP_CI != 'true' } }
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

        // Update manifests (bonus GitOps) : remplace le tag des images dans les
        // Deployments par le numéro de ce build, puis committe et pousse sur main.
        // Argo CD voit le commit et déploie la nouvelle version tout seul.
        // MySQL n'est pas concerné : il utilise l'image officielle mysql:8.4.
        stage('Update manifests') {
            when { expression { env.SKIP_CI != 'true' } }
            steps {
                // sed remplace tout ce qui suit « nom-de-l-image: » par le nouveau tag.
                sh 'sed -i "s|$REGISTRY/bijouterie-frontend:.*|$REGISTRY/bijouterie-frontend:$BUILD_NUMBER|" k8s/09-frontend-deployment.yaml'
                sh 'sed -i "s|$REGISTRY/bijouterie-backend:.*|$REGISTRY/bijouterie-backend:$BUILD_NUMBER|" k8s/06-backend-deployment.yaml'
                // Identité de l'auteur des commits faits par Jenkins.
                sh 'git config user.name "Jenkins"'
                sh 'git config user.email "jenkins@bijouterie.local"'
                sh 'git add k8s/06-backend-deployment.yaml k8s/09-frontend-deployment.yaml'
                // [skip ci] : le build lancé par ce commit s'arrêtera après le Checkout.
                sh 'git commit -m "Mise à jour des images : build $BUILD_NUMBER [skip ci]"'
                // Le token GitHub vient du credential github-credentials, jamais écrit ici.
                // HEAD:main : Jenkins travaille sur un commit détaché, on pousse vers main.
                withCredentials([usernamePassword(credentialsId: 'github-credentials',
                                                  usernameVariable: 'GIT_USER',
                                                  passwordVariable: 'GIT_TOKEN')]) {
                    sh 'git push https://$GIT_USER:$GIT_TOKEN@$GIT_REPO HEAD:main'
                }
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
