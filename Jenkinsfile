pipeline {

    agent any

    environment {

        IMAGE_NAME = "omprakash2929/appwrite"
        IMAGE_TAG  = "${BUILD_NUMBER}"

        NAMESPACE  = "appwrite"
    }

    options {

        timeout(time: 30, unit: 'MINUTES')

        buildDiscarder(
            logRotator(numToKeepStr: '10')
        )

        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {

            steps {

                echo "📥 Checking out repository..."

                checkout scm
            }
        }

        stage('Verify Environment') {

            steps {

                sh '''
                    docker --version
                '''
            }
        }

        stage('Pull Appwrite Image') {

            steps {

                sh '''
                    docker pull appwrite/appwrite:latest

                    docker tag \
                    appwrite/appwrite:latest \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Security Scan') {

            steps {
		echo "security scan..."
            }
        }

        stage('DockerHub Login') {

            steps {
		echo "docker login"
            }
        }

        stage('Push Image') {

            steps {
		echo "Doker push image..."
            }
        }

        stage('Deploy Kubernetes') {

            steps {
		echo "K8s Deploye "
            }
        }

        stage('Verify Deployment') {

            steps {
		
		echo "docker verfiy"
            }
        }
    }

    post {

        success {

            echo '✅ Pipeline Success'
        }

        failure {

            echo '❌ Pipeline Failed'
        }

        always {

            sh 'docker logout || true'

            cleanWs()
        }
    }
}
