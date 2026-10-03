pipeline {
    agent any

    environment {
        IMAGE_NAME = 'devops-aws-app'
        AWS_REGION = 'ap-south-1'
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

        stage('Push to ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-ecr-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
                        REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${REGISTRY}

                        docker tag ${IMAGE_NAME}:${BUILD_NUMBER} \
                            ${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}

                        docker push ${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}

                        docker logout ${REGISTRY}
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f smoke-test || true'
        }
    }
}
