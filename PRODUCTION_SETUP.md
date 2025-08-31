# Production EC2 Setup for Calculator App

## Quick Setup for EC2: 3.85.232.237

### 1. SSH to your production EC2
```bash
ssh -i your-key.pem ec2-user@3.85.232.237
```

### 2. Install Docker
```bash
# Update system
sudo yum update -y

# Install Docker
sudo yum install -y docker
sudo service docker start
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user

# Logout and login again
exit
ssh -i your-key.pem ec2-user@3.85.232.237
```

### 3. Install AWS CLI
```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip

# Verify installation
aws --version
```

### 4. Configure AWS (Choose ONE option)

#### Option A: Use IAM Role (Recommended)
1. Go to AWS Console → EC2 → Your instance
2. Click "Actions" → "Security" → "Modify IAM role"
3. Attach a role with ECR permissions

#### Option B: Manual AWS Configuration (Only if no IAM role)
```bash
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Enter your region (e.g., us-east-1)
```

**Note:** IAM role is more secure and recommended!

### 5. Test Docker
```bash
docker --version
docker ps
```

## What Jenkins Will Do

1. **Build** the calculator app container
2. **Push** it to ECR
3. **Deploy** to your EC2 at 3.85.232.237
4. **Health check** the deployed app

## Test Your Setup

After Jenkins deploys, test your app:
```bash
curl http://3.85.232.237/health
```

You should see: `{"status": "ok"}`

## Troubleshooting

- **Docker not running**: `sudo systemctl status docker`
- **Permission denied**: Make sure you're in docker group
- **AWS CLI not working**: Check credentials or IAM role
- **Port 80 blocked**: Check security group allows HTTP traffic
