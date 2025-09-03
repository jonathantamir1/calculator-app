pipeline {
    agent none
    
    environment {
        ECR_REGISTRY = '992382545251.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = 'jonathan-cicd'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        PRODUCTION_EC2_IP = credentials('production-ec2-ip')
    }
    
    stages {
        stage('Environment Setup') {
            agent {
                docker {
                    image 'alpine:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    echo "=== Pipeline Environment Variables ==="
                    echo "ECR_REGISTRY: ${ECR_REGISTRY}"
                    echo "ECR_REPOSITORY: ${ECR_REPOSITORY}"
                    echo "IMAGE_TAG: ${IMAGE_TAG}"
                    echo "PRODUCTION_EC2_IP: ${PRODUCTION_EC2_IP}"
                    echo "====================================="
                }
            }
        }
        
        stage('Build') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    // Build Docker image
                    sh 'docker build -t ${ECR_REPOSITORY}:${IMAGE_TAG} .'
                    sh 'docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_REPOSITORY}:latest'
                    
                    // Tag for ECR
                    sh 'docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}'
                    sh 'docker tag ${ECR_REPOSITORY}:latest ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest'
                }
            }
        }
        
        stage('Test') {
            agent {
                docker {
                    image 'python:3.11-slim'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    sh '''
                        # Install dependencies and run tests
                        pip install -r requirements.txt
                        python -m unittest discover -s tests -v
                    '''
                }
            }
        }
        
        stage('Push to ECR') {
            agent {
                docker {
                    image 'amazon/aws-cli:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh '''
                            echo "Logging into ECR..."
                            aws ecr get-login-password --region us-east-1 | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            
                            echo "Pushing images..."
                            docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            agent {
                docker {
                    image 'alpine:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    // Deploy to production EC2 using SSH credentials
                    withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key', keyFileVariable: 'SSH_KEY')]) {
                        sh '''
                            apk add --no-cache openssh-client
                            ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@${PRODUCTION_EC2_IP} '
                                # Stop existing container
                                docker stop calculator-app || true
                                docker rm calculator-app || true
                                
                                # Login to ECR (EC2 has IAM role for ECR access)
                                aws ecr get-login-password --region us-east-1 | \
                                docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                
                                # Pull latest image
                                docker pull ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                                
                                # Run new container
                                docker run -d --name calculator-app -p 80:5000 --restart unless-stopped \
                                    ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                            '
                        '''
                    }
                }
            }
        }
        
        stage('Health Check') {
            agent {
                docker {
                    image 'alpine:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    sh '''
                        apk add --no-cache curl
                        sleep 10
                        curl -f http://${PRODUCTION_EC2_IP}/health || exit 1
                        echo "Deployment successful! App is healthy."
                    '''
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Cleanup using Docker-in-Docker
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker:latest \
                        sh -c "docker rmi ${ECR_REPOSITORY}:${IMAGE_TAG} || true && \
                               docker rmi ${ECR_REPOSITORY}:latest || true && \
                               docker system prune -f || true"
                '''
            }
        }
    }
}