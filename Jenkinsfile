pipeline {
    agent any
    
    environment {
        ECR_REGISTRY = credentials('ecr-registry')
        ECR_REPOSITORY = 'calculator-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        PRODUCTION_EC2_IP = credentials('production-ec2-ip')
    }
    
        
        stage('Build') {
            steps {
                script {
                    // Build Docker image
                    sh 'docker build -t ${ECR_REPOSITORY}:${IMAGE_TAG} .'
                    sh 'docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_REPOSITORY}:latest'
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    // Run stdlib unittest inside the built image (no pip needed)
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
                        
                        // Tag and push images
                        sh 'docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}'
                        sh 'docker tag ${ECR_REPOSITORY}:latest ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest'
                        
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
                                
                                # Pull latest image (EC2 has IAM role for ECR access)
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
