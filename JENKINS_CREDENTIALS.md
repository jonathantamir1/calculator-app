# Jenkins Credentials Setup

## 🔐 No Hardcoded Secrets in Git

To keep your repository secure, you need to set up Jenkins credentials instead of hardcoding values.

### 1. Set Up Jenkins Credentials

Go to **Manage Jenkins** > **Manage Credentials** > **System** > **Global credentials** > **Add Credentials**

#### ECR Registry Credential
- **Kind**: Secret text
- **ID**: `ecr-registry`
- **Secret**: `992382545251.dkr.ecr.us-east-1.amazonaws.com/jonathan-repo`
- **Description**: ECR registry for calculator app

#### Production EC2 IP Credential
- **Kind**: Secret text
- **ID**: `production-ec2-ip`
- **Secret**: `3.85.232.237`
- **Description**: Production EC2 IP address

### 2. Alternative: Use Jenkins Environment Variables

If you prefer, you can also set these in Jenkins job configuration:

1. Go to your Jenkins job
2. Click **Configure**
3. Check **This project is parameterized**
4. Add **String Parameter**:
   - **Name**: `ECR_REGISTRY`
   - **Default Value**: `992382545251.dkr.ecr.us-east-1.amazonaws.com/jonathan-repo`
5. Add **String Parameter**:
   - **Name**: `PRODUCTION_EC2_IP`
   - **Default Value**: `3.85.232.237`

Then update the Jenkinsfile to use:
```groovy
environment {
    ECR_REGISTRY = "${params.ECR_REGISTRY}"
    ECR_REPOSITORY = 'calculator-app'
    IMAGE_TAG = "${env.BUILD_NUMBER}"
    PRODUCTION_EC2_IP = "${params.PRODUCTION_EC2_IP}"
}
```

### 3. Security Benefits

✅ **No secrets in Git** - Credentials stored securely in Jenkins  
✅ **Easy to update** - Change values without code changes  
✅ **Access control** - Jenkins manages who can see credentials  
✅ **Audit trail** - Track credential usage  

### 4. Test Your Setup

After setting up credentials, test your pipeline:
1. Run the Jenkins job
2. Check that it can access ECR
3. Verify deployment to production EC2
4. Confirm health check passes
