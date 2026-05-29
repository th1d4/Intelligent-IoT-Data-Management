pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "th1d4/iot-data-mgmt"
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
                    pip install -r requirements.txt 2>/dev/null || echo "No requirements.txt"
                    
                    # Install npm dependencies
                    cd new-frontend/frontend
                    npm install 2>/dev/null || echo "No npm dependencies"
                    cd ../..
                '''
                sh 'docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .'
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
                    trivy image --severity HIGH,CRITICAL --format json --output trivy-report.json ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
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
                    docker stop iot-staging || true
                    docker rm iot-staging || true
                    docker run -d --name iot-staging -p ${STAGING_PORT}:5000 -e ENVIRONMENT=staging ${DOCKER_IMAGE}:${BUILD_NUMBER}
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
                        docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
                sh '''
                    docker stop iot-production || true
                    docker rm iot-production || true
                    docker run -d --name iot-production -p ${PROD_PORT}:5000 -e ENVIRONMENT=production ${DOCKER_IMAGE}:latest
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
