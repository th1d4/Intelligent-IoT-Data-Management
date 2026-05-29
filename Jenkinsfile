pipeline {
    agent any

    environment {
        DOCKER_IMAGE_BACKEND = "th1d4/iot-data-mgmt-backend"
        DOCKER_IMAGE_FRONTEND = "th1d4/iot-data-mgmt-frontend"
        SONAR_PROJECT = "intelligent-iot"
        STAGING_PORT = "5001"
        PROD_PORT = "5002"
    }

    stages {
        stage('Build') {
            steps {
                echo '=== BUILD STAGE ==='
                sh '''
                    # Setup Python virtual environment
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    
                    # Use system npm
                    export PATH="/usr/local/bin:$PATH"
                    cd new-frontend/frontend
                    npm install
                    cd ../..
                    
                    # Build Docker images
                    docker build -t ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER} -f Docker/Backend-Dockerfile .
                    docker build -t ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER} -f Docker/Frontend-Dockerfile .
                '''
            }
        }

        stage('Test') {
            steps {
                echo '=== TEST STAGE ==='
                sh '''
                    . venv/bin/activate
                    pip install pytest pytest-cov
                    pytest tests/ -v --junitxml=test-results.xml --cov=. --cov-report=xml:coverage.xml || true
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results.xml'
                }
            }
        }

        stage('Code Quality') {
            steps {
                echo '=== CODE QUALITY STAGE ==='
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        docker run --rm \
                            -e SONAR_HOST_URL=${SONAR_HOST_URL} \
                            -e SONAR_TOKEN=${SONAR_AUTH_TOKEN} \
                            -v "$(pwd):/usr/src" \
                            sonarsource/sonar-scanner-cli \
                            -Dsonar.projectKey=${SONAR_PROJECT} \
                            -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Security') {
            steps {
                echo '=== SECURITY STAGE ==='
                sh '''
                    . venv/bin/activate
                    bandit -r . -f json -o bandit-report.json || true
                    trivy image --severity HIGH,CRITICAL --format json --output trivy-report.json ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER} || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bandit-report.json, trivy-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Deploy') {
            steps {
                echo '=== DEPLOY STAGE: Staging ==='
                sh '''
                    docker stop iot-staging-backend || true
                    docker rm iot-staging-backend || true
                    docker run -d --name iot-staging-backend -p ${STAGING_PORT}:5000 -e ENVIRONMENT=staging ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER}
                    
                    docker stop iot-staging-frontend || true
                    docker rm iot-staging-frontend || true
                    docker run -d --name iot-staging-frontend -p 3000:3000 -e ENVIRONMENT=staging ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Release') {
            steps {
                echo '=== RELEASE STAGE: Production ==='
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                        
                        docker tag ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER} ${DOCKER_IMAGE_BACKEND}:latest
                        docker push ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE_BACKEND}:latest
                        
                        docker tag ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER} ${DOCKER_IMAGE_FRONTEND}:latest
                        docker push ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE_FRONTEND}:latest
                    '''
                }
                sh '''
                    docker stop iot-production-backend || true
                    docker rm iot-production-backend || true
                    docker run -d --name iot-production-backend -p ${PROD_PORT}:5000 -e ENVIRONMENT=production ${DOCKER_IMAGE_BACKEND}:latest
                    
                    docker stop iot-production-frontend || true
                    docker rm iot-production-frontend || true
                    docker run -d --name iot-production-frontend -p 3001:3000 -e ENVIRONMENT=production ${DOCKER_IMAGE_FRONTEND}:latest
                '''
            }
        }

        stage('Monitoring') {
            steps {
                echo '=== MONITORING STAGE ==='
                sh '''
                    curl -f http://localhost:9090/-/healthy && echo "Prometheus OK"
                    curl -f http://localhost:3001/api/health && echo "Grafana OK"
                '''
            }
        }
    }

    post {
        success {
            echo '=== ALL STAGES COMPLETED SUCCESSFULLY ==='
        }
        failure {
            echo '=== PIPELINE FAILED ==='
        }
    }
}
