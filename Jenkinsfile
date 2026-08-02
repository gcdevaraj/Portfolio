pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "devaraj74/portfolio"
        BUILD_TAG = "${BUILD_NUMBER}"

        MANIFEST_REPO = "https://github.com/gcdevaraj/k8s-manifests.git"
        MANIFEST_BRANCH = "main"

        SONAR_SCANNER = tool 'SonarScanner'
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                    ${SONAR_SCANNER}/bin/sonar-scanner \
                    -Dsonar.projectKey=portfolio \
                    -Dsonar.projectName=portfolio \
                    -Dsonar.sources=. \
                    -Dsonar.sourceEncoding=UTF-8
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
                sh """
                docker build -t ${DOCKER_IMAGE}:${BUILD_TAG} .
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'USERNAME',
                        passwordVariable: 'PASSWORD'
                    )
                ]) {

                    sh """
                    echo \$PASSWORD | docker login -u \$USERNAME --password-stdin

                    docker push ${DOCKER_IMAGE}:${BUILD_TAG}

                    docker logout
                    """
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {

                dir('manifest') {

                    git branch: "${MANIFEST_BRANCH}",
                        url: "${MANIFEST_REPO}"

                    sh """
                    sed -i 's|image:.*|image: ${DOCKER_IMAGE}:${BUILD_TAG}|' deployment.yaml

                    git config user.email "jenkins@local"

                    git config user.name "Jenkins"

                    git add deployment.yaml

                    git commit -m "Updated image to ${BUILD_TAG}" || true

                    git push origin ${MANIFEST_BRANCH}
                    """
                }
            }
        }
    }

    post {

        success {
            echo "Pipeline Completed Successfully"
        }

        failure {
            echo "Pipeline Failed"
        }
    }
}
