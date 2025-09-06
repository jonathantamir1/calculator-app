pipeline {
    agent none
    
    environment {
        ECR_REGISTRY = '992382545251.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = 'jonathan-cicd'
        // Dynamic tagging: PR-specific for PRs, latest for master
        IMAGE_TAG = "${env.BRANCH_NAME == 'master' ? 'latest' : "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"}"
        PRODUCTION_EC2_IP = credentials('production-ec2-ip')
    }
    
    stages {
        // ========================================
        // CI/CD MULTIBRANCH PIPELINE
        // ========================================
        // CI Flow (PRs): Build → Test → Push to ECR
        // CD Flow (Master): Build → Test → Push to ECR → Deploy → Health Check
        // ========================================
        
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
                    echo "BRANCH_NAME: ${env.BRANCH_NAME}"
                    echo "CHANGE_ID: ${env.CHANGE_ID}"
                    echo "BUILD_NUMBER: ${env.BUILD_NUMBER}"
                    echo "ECR_REGISTRY: ${ECR_REGISTRY}"
                    echo "ECR_REPOSITORY: ${ECR_REPOSITORY}"
                    echo "IMAGE_TAG: ${IMAGE_TAG}"
                    echo "PRODUCTION_EC2_IP: ${PRODUCTION_EC2_IP}"
                    echo "PIPELINE_TYPE: ${env.BRANCH_NAME == 'master' ? 'CD (Deploy to Production)' : 'CI (PR Build Only)'}"
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
                    args '-v /var/run/docker.sock:/var/run/docker.sock --entrypoint=""'
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
        
        // ========================================
        // CD FLOW ONLY - Master Branch Deploy
        // ========================================
        stage('Deploy to Production') {
            when {
                branch 'master'  // Only deploy on master branch (CD flow)
            }
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
                            ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@${PRODUCTION_EC2_IP} "
                                # Stop existing container
                                docker stop calculator-app || true
                                docker rm calculator-app || true
                                
                                # Login to ECR (EC2 has IAM role for ECR access)
                                aws ecr get-login-password --region us-east-1 | \\
                                docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                
                                # Pull latest image
                                docker pull ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                                
                                # Run new container
                                docker run -d --name calculator-app -p 80:5000 --restart unless-stopped \\
                                    ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                            "
                        '''
                    }
                }
            }
        }
        
        stage('Health Check') {
            when {
                branch 'master'  // Only health check on master branch (CD flow)
            }
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
            node('docker') {
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
}