pipeline {
  agent any
  stages {
    stage('Build Docker Image') {
      steps {
        sh 'docker build -t <image_name> .'
      }
    }

    stage('Push Docker Image to ECR') {
      steps {
        withCredentials(bindings: [aws(credentialsId: 'AKIA6ODU35VRWCEWZVN5 (calc-app)', region:
                                    'us-east-1')]) {
          sh '$(aws ecr get-login --no-include-email)'
          sh 'docker push <aws_account_id>.dkr.ecr.<aws_region>.amazonaws.com/<image_name>:latest'
        }

      }
    }

    stage('Deploy Docker Container') {
      steps {
        withCredentials(bindings: [sshUserPrivateKey(credentialsId: 'ec2-user',
                                    keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
          sh 'scp -o StrictHostKeyChecking=no -i $SSH_KEY Dockerrun.aws.json ec2-user@<ec2_instance_ip>:/home/ec2-user/'
          sh 'ssh -o StrictHostKeyChecking=no -i $SSH_KEY ec2-user@<ec2_instance_ip> "docker stop <container_name> || true"'
          sh 'ssh -o StrictHostKeyChecking=no -i $SSH_KEY ec2-user@<ec2_instance_ip> "docker rm <container_name> || true"'
          sh 'ssh -o StrictHostKeyChecking=no -i $SSH_KEY ec2-user@<ec2_instance_ip> "docker run -d --name <container_name> -p 80:8080 <aws_account_id>.dkr.ecr.<aws_region>.amazonaws.com/<image_name>:latest"'
        }

      }
    }

  }
}