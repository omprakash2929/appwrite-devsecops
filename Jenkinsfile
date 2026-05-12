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

        // ─── STAGE 1 ───────────────────────────────────────
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

        // ─── STAGE 2 ───────────────────────────────────────
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

        // ─── STAGE 3 ───────────────────────────────────────
        stage('SonarQube Analysis') {
 steps {
        withSonarQubeEnv('sonar-server') {
            withCredentials([string(
                credentialsId: 'SonarQube-Token',
                variable: 'SONAR_TOKEN'
            )]) {
                sh '''/opt/sonar-scanner/bin/sonar-scanner \
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

        // ─── STAGE 4 ───────────────────────────────────────
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ─── STAGE 5 ───────────────────────────────────────
        stage('Pull Appwrite Image') {
            steps {
                sh '''
                    echo "📦 Pulling official Appwrite image..."
                    docker pull appwrite/appwrite:latest

                    echo "🏷️  Tagging image..."
                    docker tag appwrite/appwrite:latest ${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag appwrite/appwrite:latest ${IMAGE_NAME}:latest
                '''
            }
        }

        // ─── STAGE 6 ───────────────────────────────────────
        stage('Trivy Security Scan') {
            steps {
                sh '''
                    echo "🔒 Scanning image for vulnerabilities..."
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

        // ─── STAGE 7 ───────────────────────────────────────
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "☁️  Pushing to DockerHub..."
                        echo $DOCKER_PASS | docker login \
                            -u $DOCKER_USER \
                            --password-stdin

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest

                        echo "✅ Push successful!"
                        docker logout
                    '''
                }
            }
        }

        // ─── STAGE 8 ───────────────────────────────────────
        stage('Deploy to Kubernetes') {
	steps {
        sh '''
            kubectl apply -f k8s/namespace.yaml
            kubectl apply -f k8s/mongodb.yaml -n ${NAMESPACE}
            kubectl apply -f k8s/mariadb.yaml -n ${NAMESPACE}
            kubectl apply -f k8s/redis.yaml   -n ${NAMESPACE}
            
            # Wait for databases
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

        // ─── STAGE 9 ───────────────────────────────────────
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "🔍 Checking deployment status..."
                    kubectl rollout status deployment/appwrite \
                        -n ${NAMESPACE} --timeout=3m

                    echo "=== Pods ==="
                    kubectl get pods -n ${NAMESPACE}

                    echo "=== Services ==="
                    kubectl get svc -n ${NAMESPACE}
                '''
            }
        }
    }

    // ─── POST ──────────────────────────────────────────────
    post {

        success {
            echo """
            ✅ DEPLOYMENT SUCCESS
            Image : ${IMAGE_NAME}:${IMAGE_TAG}
            Author: ${env.GIT_AUTHOR}
            Commit: ${env.GIT_MSG}
            """
        }

        failure {
            echo """
            ❌ DEPLOYMENT FAILED
            Stage : ${env.STAGE_NAME}
            Author: ${env.GIT_AUTHOR}
            Build : ${BUILD_URL}
            """
        }

        always {
            sh 'docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true'
            sh 'docker logout || true'
            cleanWs()
        }
    }
}
