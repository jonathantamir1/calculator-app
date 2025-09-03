# Fix GitHub API Rate Limiting in Jenkins

## 🚨 **Error You're Seeing:**
```
Jenkins-Imposed API Limiter: Current quota for Github API usage has 52 remaining (1 over budget)
java.lang.InterruptedException: sleep interrupted
```

## 🔧 **How to Fix This:**

### **Step 1: Add GitHub Credentials to Jenkins**

1. **Go to**: Jenkins → Manage Jenkins → Manage Credentials → System → Global credentials
2. **Click**: "Add Credentials"
3. **Select**: "GitHub App" or "Personal Access Token"
4. **Fill in**:
   - **ID**: `github-credentials`
   - **Description**: GitHub credentials for API access
   - **GitHub App** or **Personal Access Token**

### **Step 2: Configure Your Multibranch Pipeline Job**

1. **Go to**: Your Calculator App job → Configure
2. **In Branch Sources section**:
   - **Source**: GitHub
   - **Credentials**: Select your `github-credentials`
   - **Repository**: `jonathantamir1/calculator-app`
   - **Behaviours**: Leave default

### **Step 3: Fix GitHub API Rate Limiting**

1. **Go to**: Manage Jenkins → Configure System
2. **Find**: "GitHub API usage" section
3. **Change**: From "Jenkins-imposed" to "GitHub-imposed"
4. **Save**

### **Step 4: Alternative - Use Personal Access Token**

If GitHub App doesn't work:

1. **Go to GitHub**: Settings → Developer settings → Personal access tokens
2. **Generate new token** with `repo` permissions
3. **In Jenkins**: Add as "Username with password" credential
   - **Username**: Your GitHub username
   - **Password**: The token

## 🧪 **Test the Fix:**

1. **Save your job configuration**
2. **Run the job again**
3. **Check**: No more rate limiting errors
4. **Verify**: Pipeline stages execute properly

## 🔍 **Why This Happens:**

- **Anonymous access**: Jenkins tries to access GitHub without credentials
- **Rate limits**: GitHub limits anonymous API calls to 60/hour
- **Jenkins-imposed**: Jenkins adds extra restrictions
- **GitHub-imposed**: Uses GitHub's actual limits (higher with credentials)

## ✅ **Expected Result:**

After fixing:
- ✅ No more rate limiting errors
- ✅ Faster branch scanning
- ✅ Higher API limits (5000/hour with authenticated user)
- ✅ Pipeline runs successfully

## 🚨 **If Still Having Issues:**

1. **Check GitHub token permissions**
2. **Verify repository access**
3. **Restart Jenkins after changes**
4. **Check Jenkins logs for other errors**

Your Jenkinsfile code is correct - this is purely a GitHub authentication issue! 🎯
