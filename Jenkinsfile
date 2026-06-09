pipeline {
    agent any

    environment {
        // ── CHANGE THESE ──────────────────────────────────────────
        DOCKER_IMAGE        = "your-dockerhub-username/java-cicd-app"
        SONAR_URL           = "http://your-sonarqube-ip:9000"
        HELM_REPO_URL       = "https://github.com/your-org/helm-charts-repo"
        HELM_REPO_NAME      = "helm-charts-repo"
        // ──────────────────────────────────────────────────────────

        DOCKER_TAG          = "${BUILD_NUMBER}"
        DOCKER_CREDS        = credentials('dockerhub-creds')     // Jenkins credential ID
        GIT_TOKEN           = credentials('github-token')        // Jenkins credential ID
    }

    stages {

        stage('Checkout') {
            steps {
                echo '>>> Checking out source code...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo '>>> Building application with Maven...'
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                echo '>>> Running unit tests...'
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '>>> Running SonarQube code quality analysis...'
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=java-cicd-app \
                          -Dsonar.host.url=${SONAR_URL} \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo '>>> Waiting for SonarQube Quality Gate...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo ">>> Building Docker image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
                echo '>>> Pushing Docker image to DockerHub...'
                sh """
                    echo ${DOCKER_CREDS_PSW} | docker login -u ${DOCKER_CREDS_USR} --password-stdin
                    docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                    docker push ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Update Helm Chart') {
            steps {
                echo '>>> Updating Helm chart with new image tag...'
                sh """
                    rm -rf ${HELM_REPO_NAME}
                    git clone https://${GIT_TOKEN}@github.com/your-org/${HELM_REPO_NAME}.git
                    cd ${HELM_REPO_NAME}

                    # Update image tag in values.yaml
                    sed -i "s|tag:.*|tag: \\"${DOCKER_TAG}\\"|" values.yaml

                    git config user.email "jenkins@cicd.com"
                    git config user.name "Jenkins CI"
                    git add values.yaml
                    git commit -m "ci: update image tag to ${DOCKER_TAG} [skip ci]"
                    git push origin main
                """
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully! ArgoCD will auto-deploy.'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs above.'
        }
        always {
            // Clean up docker images to save space
            sh "docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"
            sh "docker rmi ${DOCKER_IMAGE}:latest || true"
        }
    }
}
