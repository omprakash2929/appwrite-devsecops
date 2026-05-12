pipeline {

    agent any

    environment {
        IMAGE_NAME    = "omprakash2929/appwrite"
        IMAGE_TAG     = "${BUILD_NUMBER}"
        NAMESPACE     = "appwrite"
        SONAR_PROJECT = "appwrite-devsecops"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {

        // ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo "📥 Checking out repository..."
                checkout scm

                script {
                    env.GIT_AUTHOR = sh(
                        script: 'git log -1 --pretty=%an',
                        returnStdout: true
                    ).trim()

                    env.GIT_MSG = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()
                }

                echo "Author : ${env.GIT_AUTHOR}"
                echo "Commit : ${env.GIT_MSG}"
            }
        }

        // ─────────────────────────────────────────────
        stage('Verify Environment') {
            steps {
                sh '''
                    echo "=== Docker ==="
                    docker --version

                    echo "=== Kubectl ==="
                    kubectl version --client

                    echo "=== Helm ==="
                    helm version

                    echo "=== Trivy ==="
                    trivy --version
                '''
            }
        }

        // ─────────────────────────────────────────────
        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('sonar-server') {

                    withCredentials([
                        string(
                            credentialsId: 'SonarQube-Token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {

                        sh '''
                            /opt/sonar-scanner/bin/sonar-scanner \
                            -Dsonar.projectKey=appwrite-devsecops \
                            -Dsonar.projectName="Appwrite DevSecOps" \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/vendor/**,**/node_modules/** \
                            -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
            }
        }

        // ─────────────────────────────────────────────
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ─────────────────────────────────────────────
        stage('Pull Appwrite Image') {
            steps {

                sh '''
                    echo "📦 Pulling official Appwrite image..."

                    docker pull appwrite/appwrite:latest

                    echo "🏷️ Tagging image..."

                    docker tag appwrite/appwrite:latest ${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag appwrite/appwrite:latest ${IMAGE_NAME}:latest
                '''
            }
        }

        // ─────────────────────────────────────────────
        stage('Trivy Security Scan') {

            steps {

                sh '''
                    echo "🔒 Running Trivy Scan..."

                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    --format table \
                    --no-progress \
                    ${IMAGE_NAME}:${IMAGE_TAG} | tee trivy-report.txt
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt'
                }
            }
        }

        // ─────────────────────────────────────────────
        stage('Push to DockerHub') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                        echo "☁️ Pushing image to DockerHub..."

                        echo $DOCKER_PASS | docker login \
                        -u $DOCKER_USER \
                        --password-stdin

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest

                        docker logout

                        echo "✅ Docker push completed!"
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────
        stage('Deploy to Kubernetes') {

            steps {

                sh '''
                    kubectl apply -f k8s/namespace.yaml

                    kubectl apply -f k8s/mongodb.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/mariadb.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/redis.yaml -n ${NAMESPACE}

                    echo "⏳ Waiting for DB services..."
                    sleep 30

                    helm upgrade --install appwrite ./helm/appwrite \
                    --namespace ${NAMESPACE} \
                    --create-namespace \
                    --set image.repository=${IMAGE_NAME} \
                    --set image.tag=${IMAGE_TAG} \
                    --wait \
                    --timeout 5m \
                    --atomic
                '''
            }
        }

        // ─────────────────────────────────────────────
        stage('Verify Deployment') {

            steps {

                sh '''
                    echo "🔍 Checking deployment..."

                    kubectl rollout status deployment/appwrite \
                    -n ${NAMESPACE} --timeout=3m

                    echo "=== Pods ==="
                    kubectl get pods -n ${NAMESPACE}

                    echo "=== Services ==="
                    kubectl get svc -n ${NAMESPACE}
                '''
            }
        }

        // ─────────────────────────────────────────────
        stage('Deploy Monitoring') {

            steps {

                sh '''
                    echo "📊 Deploying monitoring resources..."

                    kubectl apply -f monitoring/servicemonitor.yaml
                    kubectl apply -f monitoring/alert-rules.yaml

                    echo "✅ Monitoring deployment completed!"
                '''
            }
        }

        // ─────────────────────────────────────────────
        stage('Verify Monitoring') {

            steps {

                sh '''
                    echo "🔍 Verifying monitoring resources..."

                    kubectl get servicemonitor -n monitoring
                    kubectl get prometheusrule -n monitoring
                '''
            }
        }
    }

    // ─────────────────────────────────────────────

post {

    success {

        echo """
        ✅ DEPLOYMENT SUCCESS

        Image  : ${IMAGE_NAME}:${IMAGE_TAG}
        Author : ${env.GIT_AUTHOR}
        Commit : ${env.GIT_MSG}
        """

        withCredentials([
            string(
                credentialsId: 'slack-webhook',
                variable: 'SLACK_URL'
            )
        ]) {

            sh """
                curl -X POST -H 'Content-type: application/json' \
                --data '{
                    "text":"✅ *Appwrite Deployment Successful*\\n🚀 Build: #${BUILD_NUMBER}\\n📦 Image: ${IMAGE_NAME}:${IMAGE_TAG}\\n👤 Author: ${env.GIT_AUTHOR}\\n📝 Commit: ${env.GIT_MSG}\\n🔗 Build URL: ${BUILD_URL}"
                }' \$SLACK_URL
            """
        }
    }

    failure {

        echo """
        ❌ DEPLOYMENT FAILED

        Stage  : ${env.STAGE_NAME}
        Author : ${env.GIT_AUTHOR}
        Build  : ${BUILD_URL}
        """

        withCredentials([
            string(
                credentialsId: 'slack-webhook',
                variable: 'SLACK_URL'
            )
        ]) {

            sh """
                curl -X POST -H 'Content-type: application/json' \
                --data '{
                    "text":"❌ *Appwrite Deployment Failed*\\n🚨 Stage: ${env.STAGE_NAME}\\n👤 Author: ${env.GIT_AUTHOR}\\n🔗 Build URL: ${BUILD_URL}"
                }' \$SLACK_URL
            """
        }
    }

    always {

        sh 'docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true'
        sh 'docker logout || true'

        cleanWs()
    }
}
