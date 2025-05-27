pipeline {
    agent any

    environment {
        // Frontend
        DOCKER_IMAGE = "tebraouisamy/presence-frontend"
        // Backend
        BACKEND_DOCKER_IMAGE = "tebraouisamy/presence-backend"

        DOCKER_CREDENTIALS_ID = "docker-credentials"
        KUBECONFIG_ID = "kubeconfig"
        NAMESPACE = "presence-app"

        // ⚠️ Pour éviter crash SIGBUS (Next.js)
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

        stage('Build Frontend Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Push Frontend Docker Image') {
            steps {
                script {
                    docker.withRegistry("https://index.docker.io/v1/", "${DOCKER_CREDENTIALS_ID}") {
                        docker.image("${DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                    }
                }
            }
        }

        stage('Build Backend Docker Image') {
            steps {
                script {
                    docker.build("${BACKEND_DOCKER_IMAGE}:${BUILD_NUMBER}", "./backend")
                }
            }
        }

        stage('Push Backend Docker Image') {
            steps {
                script {
                    docker.withRegistry("https://index.docker.io/v1/", "${DOCKER_CREDENTIALS_ID}") {
                        docker.image("${BACKEND_DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                    }
                }
            }
        }

        stage('Deploy Frontend to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: "${KUBECONFIG_ID}"]) {
                    sh '''
                        echo "🚀 Déploiement Frontend..."
                        kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}
                        cat kubernetes/frontend-deployment.yaml | envsubst | kubectl apply -f -
                        kubectl apply -f kubernetes/frontend-service.yaml
                        kubectl apply -f kubernetes/ingress.yaml
                    '''
                }
            }
        }

        stage('Verify Frontend Deployment') {
            steps {
                withKubeConfig([credentialsId: "${KUBECONFIG_ID}"]) {
                    sh "kubectl rollout status deployment/frontend -n ${NAMESPACE}"
                }
            }
        }

        stage('Deploy Backend to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: "${KUBECONFIG_ID}"]) {
                    sh '''
                        echo "🚀 Déploiement Backend..."
                        kubectl apply -f kubernetes/backend-deployment.yaml
                        kubectl apply -f kubernetes/backend-service.yaml
                    '''
                }
            }
        }

        stage('Verify Backend Deployment') {
            steps {
                withKubeConfig([credentialsId: "${KUBECONFIG_ID}"]) {
                    sh "kubectl rollout status deployment/backend -n ${NAMESPACE}"
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
