pipeline {
    agent any

    environment {
        IMAGE_NAME = 'devops-aws-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install -r requirements-dev.txt
                    pytest -v
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build --pull -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh 'trivy image --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ${IMAGE_NAME}:${BUILD_NUMBER}'
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    docker rm -f smoke-test || true
                    docker run -d --name smoke-test -p 5001:5000 ${IMAGE_NAME}:${BUILD_NUMBER}
                    for i in $(seq 1 15); do
                        status=$(docker inspect --format='{{.State.Health.Status}}' smoke-test)
                        [ "$status" = "healthy" ] && break
                        sleep 2
                    done
                    [ "$status" = "healthy" ]
                    curl -f http://localhost:5001/health
                '''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f smoke-test || true'
        }
    }
}
