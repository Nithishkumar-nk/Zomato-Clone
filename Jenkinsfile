pipeline {
    agent any

    tools {
        nodejs 'node20'
    }

    environment {
        DOCKER_IMAGE = 'nithishkumar24/zomato-clone'
        SONAR_PROJECT_KEY = 'zomato-clone'
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📥 Checking out source code...'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                echo '📦 Installing Node.js dependencies...'

                sh '''
                    node --version
                    npm --version
                    npm ci
                '''
            }
        }

        stage('Unit Test') {
            steps {
                echo '🧪 Running unit tests...'

                sh '''
                    CI=true npm test -- --watchAll=false --passWithNoTests
                '''
            }
        }

        stage('Build') {
            steps {
                echo '🏗️ Building React application...'

                sh '''
                    npm run build
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    echo '🔍 Running SonarQube analysis...'

                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.projectName="Zomato Clone" \
                              -Dsonar.sources=src
                        """
                    }
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                echo '🛡️ Scanning source code with Trivy...'

                sh '''
                    trivy fs \
                      --config /dev/null \
                      --scanners vuln,secret \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      .
                '''
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                script {
                    echo '🔐 Running OWASP Dependency-Check...'

                    def dependencyCheckHome = tool 'dependency-check'

                    sh """
                        ${dependencyCheckHome}/bin/dependency-check.sh \
                          --project "Zomato Clone" \
                          --scan . \
                          --format HTML \
                          --format XML \
                          --out dependency-check-report \
                          --failOnCVSS 11
                    """
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo '🐳 Building Docker image...'

                sh '''
                    docker build \
                      -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                      .

                    docker tag \
                      ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                      ${DOCKER_IMAGE}:latest
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo '🛡️ Scanning Docker image with Trivy...'

                sh '''
                    trivy image \
                      --config /dev/null \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      ${DOCKER_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Docker Hub Push') {
            steps {
                echo '📤 Pushing Docker image to Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                          -u "$DOCKER_USERNAME" \
                          --password-stdin

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        docker push ${DOCKER_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '✅ Zomato DevSecOps pipeline completed successfully!'
        }

        failure {
            echo '❌ Pipeline failed. Check the failed stage and console output.'
        }

        always {
            echo '🧹 Cleaning unused Docker images...'

            sh '''
                docker image prune -f || true
            '''
        }
    }
}
