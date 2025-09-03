pipeline {
    agent any
    
    environment {
        ECR_REGISTRY = '992382545251.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = 'jonathan-cicd'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        PRODUCTION_EC2_IP = credentials('production-ec2-ip')
    }
    
    stages {
        stage('Environment Setup') {
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
            steps {
                script {
                    sh 'docker run --rm ${ECR_REPOSITORY}:${IMAGE_TAG} python -m unittest discover -s tests -v'
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    // Login to ECR using AWS credentials
                    withCredentials([aws(credentials: 'aws-credentials', region: 'us-east-1')]) {
                        sh '''
                            aws ecr get-login-password --region us-east-1 | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        '''
                        
                        // Push images (already tagged in Build stage)
                        sh 'docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}'
                        sh 'docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest'
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            steps {
                script {
                    // Deploy to production EC2 using SSH credentials
                    withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key', keyFileVariable: 'SSH_KEY')]) {
                        sh '''
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
            steps {
                script {
                    // Wait for deployment
                    sh 'sleep 10'
                    
                    // Health check
                    sh 'curl -f http://${PRODUCTION_EC2_IP}/health || exit 1'
                    echo 'Deployment successful! App is healthy.'
                }
            }
        }
    }
    
    post {
        always {
            // Cleanup
            sh 'docker rmi ${ECR_REPOSITORY}:${IMAGE_TAG} || true'
            sh 'docker rmi ${ECR_REPOSITORY}:latest || true'
            sh 'docker system prune -f || true'
        }
    }
}