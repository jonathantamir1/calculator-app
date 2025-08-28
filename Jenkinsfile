pipeline {
    agent any

    stages {
		stage('Checkout') {
		  steps {
			checkout scm
			sh 'git rev-parse --short HEAD > .git-commit'
			script {
			  env.GIT_COMMIT_SHORT = readFile('.git-commit').trim()
			}
		  }
		}

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t <image_name> .'
            }
        }
        stage('Push Docker Image to ECR') {
            steps {
                withCredentials([aws(credentialsId: '<credentials_id>', region:
                    '<aws_region>')]) {
                    sh '$(aws ecr get-login --no-include-email)'
                    sh 'docker push <aws_account_id>.dkr.ecr.<aws_region>.amazonaws.com/<image_name>:latest'
                }
            }
        }
        stage('Deploy Docker Container') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: '<credentials_id>',
                    keyFileVariable: 'SSH_KEY_FILE')]) {
                    sh 'scp -o StrictHostKeyChecking=no -i $SSH_KEY_FILE Dockerrun.aws.json ec2-user@<ec2_instance_ip>:/home/ec2-user/'
                    sh 'ssh -o StrictHostKeyChecking=no -i $SSH_KEY_FILE ec2-user@<ec2_instance_ip> "docker stop <container_name> || true"'
                    sh 'ssh -o StrictHostKeyChecking=no -i $SSH_KEY_FILE ec2-user@<ec2_instance_ip> "docker rm <container_name> || true"'
                    sh 'ssh -o StrictHostKeyChecking=no -i $SSH_KEY_FILE ec2-user@<ec2_instance_ip> "docker run -d --name <container_name> -p 80:8080 <aws_account_id>.dkr.ecr.<aws_region>.amazonaws.com/<image_name>:latest"'
                }
            }
        }
    }
}