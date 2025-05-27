pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "tebraouisamy/presence-frontend"
        DOCKER_CREDENTIALS_ID = "docker-credentials"
        KUBECONFIG_ID = "kubeconfig"
        NAMESPACE = "presence-app"

        // Ajouts importants pour éviter SIGBUS
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

        stage('Install Frontend & Build') {
            steps {
                sh '''
                    mkdir -p .next-cache
                    chmod -R 777 .next-cache
                    npm install --legacy-peer-deps
                    npm run build
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test || true'
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
                    sh "cat kubernetes/frontend-deployment.yaml | envsubst | kubectl apply -f -"
                    sh "kubectl apply -f kubernetes/frontend-service.yaml"
                    sh "kubectl apply -f kubernetes/ingress.yaml"
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
    }
}
