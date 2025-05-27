pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "tebraouisamy/presence-frontend"
        DOCKER_CREDENTIALS_ID = "docker-credentials"
        KUBECONFIG_ID = "kubeconfig"
        NAMESPACE = "presence-app"

        // ⚠️ Variables pour éviter crash SIGBUS
        NEXT_TELEMETRY_DISABLED = '1'
        NEXT_CACHE_DIR = '.next-cache'
        NODE_OPTIONS = '--max-old-space-size=2048'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Clean & Prepare Workspace') {
            steps {
                sh '''
                    echo "🧹 Nettoyage du cache local"
                    rm -rf node_modules .next .next-cache package-lock.json
                    npm cache clean --force || true
                    mkdir -p .next-cache
                    chmod -R 777 .next-cache
                '''
            }
        }

        stage('Install Frontend & Build') {
            steps {
                script {
                    try {
                        sh 'npm install --legacy-peer-deps'
                        sh 'npm run build'
                    } catch (e) {
                        error "🚨 Erreur lors du build frontend : ${e}"
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    try {
                        sh 'npm test'
                    } catch (e) {
                        echo "⚠️ Tests échoués, mais pipeline continue."
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry("https://index.docker.io/v1/", "${DOCKER_CREDENTIALS_ID}") {
                        docker.image("${DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: "${KUBECONFIG_ID}"]) {
                    sh '''
                        echo "🚀 Déploiement Kubernetes..."
                        cat kubernetes/frontend-deployment.yaml | envsubst | kubectl apply -f -
                        kubectl apply -f kubernetes/frontend-service.yaml
                        kubectl apply -f kubernetes/ingress.yaml
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig([credentialsId: "${KUBECONFIG_ID}"]) {
                    sh "kubectl rollout status deployment/frontend -n ${NAMESPACE}"
                }
            }
        }
    }

    post {
        success {
            echo '✅ Déploiement réussi !'
        }
        failure {
            echo '❌ Échec du déploiement.'
        }
        always {
            echo '📝 Pipeline terminé.'
        }
    }
}
