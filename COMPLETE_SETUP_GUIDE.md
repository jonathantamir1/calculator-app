# Complete Jenkins Pipeline Setup Guide

## 🎯 **What We're Building:**

A complete CI/CD pipeline that:
1. **Builds** your calculator app Docker image
2. **Tests** the application
3. **Pushes** to AWS ECR
4. **Deploys** to production EC2
5. **Verifies** deployment with health checks

## 🔧 **Step-by-Step Setup:**

### **Step 1: Fix Jenkins Job Configuration**

1. **Go to**: Your Jenkins job → Configure
2. **In Branch Sources section**:
   - **Source**: GitHub
   - **Credentials**: Leave as "None" (since repo is public)
   - **Repository**: `jonathanamir1/calculator-app`
   - **Behaviours**: Leave as default
3. **Save** configuration

### **Step 2: Set Up Jenkins Credentials**

Go to **Manage Jenkins** → **Manage Credentials** → **System** → **Global credentials**

#### **Add These Credentials:**

1. **AWS Credentials**:
   - **Kind**: AWS Credentials
   - **ID**: `aws-credentials`
   - **Access Key ID**: Your AWS Access Key
   - **Secret Access Key**: Your AWS Secret Key
   - **Description**: AWS credentials for ECR access

2. **ECR Registry**:
   - **Kind**: Secret text
   - **ID**: `ecr-registry`
   - **Secret**: `992382545251.dkr.ecr.us-east-1.amazonaws.com/jonathan-repo`
   - **Description**: ECR registry URL

3. **Production EC2 IP**:
   - **Kind**: Secret text
   - **ID**: `production-ec2-ip`
   - **Secret**: `3.85.232.237`
   - **Description**: Production EC2 IP address

4. **SSH Key**:
   - **Kind**: SSH Username with private key
   - **ID**: `ssh-key`
   - **Username**: `ec2-user`
   - **Private Key**: Your SSH private key (`.pem` file content)
   - **Description**: SSH key for production EC2 access

### **Step 3: Fix Docker Socket Mounting**

SSH to your Jenkins EC2 and update docker-compose.yml:

```bash
cd Jenkins-ec2-setup
nano docker-compose.yml
```

**Add these lines to your jenkins service:**

```yaml
volumes:
  - jenkins_home:/var/jenkins_home
  - /var/run/docker.sock:/var/run/docker.sock    # ← ADD THIS
  - /usr/bin/docker:/usr/bin/docker              # ← ADD THIS

user: root                                        # ← ADD THIS
group_add:                                        # ← ADD THIS
  - docker                                       # ← ADD THIS
```

**Restart Jenkins:**
```bash
docker-compose down
docker-compose up -d
```

### **Step 4: Set Up Production EC2**

SSH to your production EC2 (3.85.232.237):

```bash
# Install Docker
sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip

# Logout and login again
exit
ssh -i your-key.pem ec2-user@3.85.232.237
```

### **Step 5: Configure IAM Role for Production EC2**

1. **Go to AWS Console** → EC2 → Your production instance
2. **Click "Actions"** → "Security" → "Modify IAM role"
3. **Attach role** with these permissions:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "ecr:GetAuthorizationToken",
                   "ecr:BatchCheckLayerAvailability",
                   "ecr:GetDownloadUrlForLayer",
                   "ecr:BatchGetImage"
               ],
               "Resource": "*"
           }
       ]
   }
   ```

### **Step 6: Test Your Setup**

1. **In Jenkins**: Scan Multibranch Pipeline Now
2. **Run the pipeline**
3. **Check each stage**:
   - Environment Setup: Should show all variables
   - Build: Should create Docker image
   - Test: Should run tests
   - Push to ECR: Should push to AWS
   - Deploy: Should deploy to EC2
   - Health Check: Should verify deployment

## 🚨 **Common Issues and Fixes:**

### **Issue 1: Git Checkout Fails**
- **Solution**: Remove credentials requirement (public repo)
- **Check**: Repository URL is correct

### **Issue 2: Docker Build Fails**
- **Solution**: Fix Docker socket mounting
- **Check**: Docker service is running on Jenkins EC2

### **Issue 3: ECR Push Fails**
- **Solution**: Verify AWS credentials
- **Check**: ECR repository exists and is accessible

### **Issue 4: SSH Deploy Fails**
- **Solution**: Verify SSH key and EC2 access
- **Check**: Security groups allow SSH from Jenkins EC2

### **Issue 5: Health Check Fails**
- **Solution**: Check EC2 security group allows HTTP (port 80)
- **Check**: Container is running and accessible

## ✅ **Verification Checklist:**

- [ ] Jenkins job configured without credentials (public repo)
- [ ] All Jenkins credentials set up
- [ ] Docker socket mounted in Jenkins container
- [ ] Production EC2 has Docker and AWS CLI
- [ ] Production EC2 has proper IAM role
- [ ] Security groups configured correctly
- [ ] Pipeline runs without errors
- [ ] Application deploys successfully
- [ ] Health check passes

## 🎉 **Expected Result:**

After completing all steps:
- ✅ **Pipeline runs successfully** from start to finish
- ✅ **Docker images built** and pushed to ECR
- ✅ **Application deployed** to production EC2
- ✅ **Health checks pass** confirming deployment
- ✅ **Full CI/CD pipeline** working end-to-end

## 🚀 **Next Steps:**

1. **Follow each step** in order
2. **Test after each major step**
3. **If issues arise**, check the troubleshooting section
4. **Verify final result** with a complete pipeline run

Your pipeline will be production-ready! 🎯
