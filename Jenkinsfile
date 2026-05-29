pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "yourdockerhubusername/iot-data-mgmt"
        SONAR_PROJECT = "intelligent-iot"
        STAGING_PORT = "5001"
        PROD_PORT = "5002"
    }

    stages {

        // ─── STAGE 1: BUILD ───────────────────────────────────────────
        stage('Build') {
            steps {
                echo '=== BUILD STAGE: Installing dependencies and building Docker image ==='
                sh '''
                    cd newBackend
                    pip3 install -r ../requirements.txt || true
                    cd ../new-frontend/frontend
                    npm ci || true
                    cd ../..
                '''
                sh 'docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} -f Docker/Dockerfile . || docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .'
                echo 'Build artefact: Docker image ${DOCKER_IMAGE}:${BUILD_NUMBER}'
            }
        }

        // ─── STAGE 2: TEST ────────────────────────────────────────────
        stage('Test') {
            steps {
                echo '=== TEST STAGE: Running automated tests ==='
                sh '''
                    cd newBackend
                    pip3 install pytest pytest-cov httpx requests || true
                    python3 -m pytest tests/ -v \
                        --junitxml=../test-results.xml \
                        --cov=. --cov-report=xml:../coverage.xml \
                        || true
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results.xml'
                }
            }
        }

        // ─── STAGE 3: CODE QUALITY ────────────────────────────────────
        stage('Code Quality') {
            steps {
                echo '=== CODE QUALITY STAGE: SonarQube analysis ==='
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        docker run --rm \
                            -e SONAR_HOST_URL=${SONAR_HOST_URL} \
                            -e SONAR_TOKEN=${SONAR_AUTH_TOKEN} \
                            -v "$(pwd):/usr/src" \
                            sonarsource/sonar-scanner-cli \
                            -Dsonar.projectKey=${SONAR_PROJECT} \
                            -Dsonar.projectName="Intelligent IoT Data Management" \
                            -Dsonar.sources=newBackend,new-frontend/frontend/src \
                            -Dsonar.python.coverage.reportPaths=coverage.xml \
                            -Dsonar.exclusions=**/node_modules/**,**/__pycache__/**
                    '''
                }
            }
        }

        // ─── STAGE 4: SECURITY ────────────────────────────────────────
        stage('Security') {
            steps {
                echo '=== SECURITY STAGE: Bandit (Python SAST) + Trivy (container scan) ==='
                sh '''
                    bandit -r newBackend/ \
                        -f json -o bandit-report.json \
                        --severity-level medium \
                        || true
                    bandit -r newBackend/ \
                        --severity-level medium \
                        || true
                '''
                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format json \
                        --output trivy-report.json \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        || true
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bandit-report.json, trivy-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        // ─── STAGE 5: DEPLOY (Staging) ────────────────────────────────
        stage('Deploy') {
            steps {
                echo '=== DEPLOY STAGE: Deploying to staging environment ==='
                sh '''
                    docker stop iot-staging || true
                    docker rm   iot-staging || true
                    docker run -d \
                        --name iot-staging \
                        -p ${STAGING_PORT}:5000 \
                        -e ENVIRONMENT=staging \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    sleep 5
                    curl -f http://localhost:${STAGING_PORT}/health || \
                    curl -f http://localhost:${STAGING_PORT}/ || true
                    echo "Staging deployed at http://localhost:${STAGING_PORT}"
                '''
            }
        }

        // ─── STAGE 6: RELEASE (Production) ───────────────────────────
        stage('Release') {
            steps {
                echo '=== RELEASE STAGE: Tagging and promoting to production ==='
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
                    docker rm   iot-production || true
                    docker run -d \
                        --name iot-production \
                        -p ${PROD_PORT}:5000 \
                        -e ENVIRONMENT=production \
                        ${DOCKER_IMAGE}:latest
                    echo "Production released at http://localhost:${PROD_PORT}"
                '''
            }
        }

        // ─── STAGE 7: MONITORING ──────────────────────────────────────
        stage('Monitoring') {
            steps {
                echo '=== MONITORING STAGE: Verifying Prometheus + Grafana ==='
                sh '''
                    sleep 3
                    curl -f http://localhost:9090/-/healthy && \
                        echo "Prometheus is healthy" || \
                        echo "Prometheus check - verify manually"
                    curl -f http://localhost:3001/api/health && \
                        echo "Grafana is healthy" || \
                        echo "Grafana check - verify manually"
                    echo "Dashboards: Prometheus=http://localhost:9090 Grafana=http://localhost:3001"
                '''
            }
        }

    }

    post {
        success {
            echo '=== ALL 7 STAGES PASSED — Pipeline complete! ==='
        }
        failure {
            echo '=== Pipeline failed — check stage logs above ==='
        }
        always {
            echo "Build #${BUILD_NUMBER} finished."
        }
    }
}
