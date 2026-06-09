pipeline {
    agent any

    environment {
        DOCKER_IMAGE   = "nimnaevz96/java-cicd-app"
        SONAR_URL      = "http://54.238.224.46:9000"
        HELM_REPO_NAME = "helm-charts-repo"
        DOCKER_TAG     = "${BUILD_NUMBER}"
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
                echo '>>> Running SonarQube analysis...'
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=java-cicd-app \
                          -Dsonar.host.url=${SONAR_URL}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Update Helm Chart') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh """
                        rm -rf ${HELM_REPO_NAME}
                        git clone https://${GIT_USER}:${GIT_PASS}@github.com/nimnadev/${HELM_REPO_NAME}.git
                        cd ${HELM_REPO_NAME}
                        sed -i "s|tag:.*|tag: \\"${DOCKER_TAG}\\"|" values.yaml
                        git config user.email "jenkins@cicd.com"
                        git config user.name "Jenkins CI"
                        git add values.yaml
                        git commit -m "ci: update image tag to ${DOCKER_TAG}"
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed! ArgoCD will auto-deploy.'
        }
        failure {
            echo '❌ Pipeline failed. Check logs above.'
        }
    }
}