# Flask Application CI/CD Deployment Guide

## Part 1: Initial Server Setup

### Step 1: Create Application Directory
```bash
mkdir -p /home/ec2-user/app
cd /home/ec2-user/app
```

### Step 2: Set Up Python Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate
```

The virtual environment isolates your Flask application's dependencies from other applications running on the server.

### Step 3: Create Systemd Service File

Create `/etc/systemd/system/flask-app.service`:

```ini
[Unit]
Description=My Flask App
After=network.target

[Service]
User=ec2-user
Group=ec2-user
WorkingDirectory=/home/ec2-user/app
Environment="PATH=/home/ec2-user/app/venv/bin"
ExecStart=/home/ec2-user/app/venv/bin/python app.py

[Install]
WantedBy=multi-user.target
```

**Configuration Breakdown:**
- **[Unit] section**: Provides service description and ensures network is ready before starting
- **[Service] section**: 
  - User/Group: Specifies which account runs the service (ec2-user for AWS Linux AMI)
  - WorkingDirectory: Path to your Flask app
  - Environment PATH: Points to the virtual environment's Python interpreter
  - ExecStart: Command to launch your application
- **[Install] section**: multi-user.target means the service starts during system boot in multi-user mode

### Step 4: Enable and Start the Service
```bash
sudo systemctl daemon-reload
sudo systemctl enable flask-app.service
sudo systemctl start flask-app.service
```

### Step 5: Configure Firewall/Security Group
Allow inbound traffic on port 5000 (or your application's port):
- AWS Security Group: Add inbound rule for port 5000
- Linux firewall: `sudo firewall-cmd --add-port=5000/tcp --permanent`

---

## Part 2: Jenkins CI/CD Pipeline

### Pipeline Stages Overview

The complete pipeline follows this flow:

1. **Checkout** → Get code from Git repository
2. **Install Dependencies** → Run `pip install -r requirements.txt`
3. **Test** → Execute `pytest` for automated testing
4. **Package** → Create a zip file of application code
5. **Deploy to Production** → Copy code to server and restart application

### Jenkins Credentials Setup

You need two credentials:

1. **SSH Key Credential**: The private key file (e.g., main.pem) for authenticating to your production server
2. **Server IP Credential**: The production server's IP address (stored separately so it's not hardcoded)

### Jenkinsfile Example

```groovy
pipeline {
    agent any
    
    environment {
        SERVER_IP = credentials('prod-server-ip')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }
        
        stage('Test') {
            steps {
                sh 'pytest'
            }
        }
        
        stage('Package') {
            steps {
                sh 'zip -r my_app.zip . -x ".git/*"'
                sh 'ls -la my_app.zip'
            }
        }
        
        stage('Deploy to Production') {
            steps {
                withCredentials([file(credentialsId: 'ssh-key', variable: 'SSH_KEY'), 
                                usernamePassword(credentialsId: 'server-user', 
                                                usernameVariable: 'USERNAME', 
                                                passwordVariable: 'PASSWORD')]) {
                    sh '''
                        # Copy zip file to production server
                        scp -i $SSH_KEY -o StrictHostKeyChecking=no \
                            my_app.zip $USERNAME@$SERVER_IP:/home/ec2-user/
                        
                        # SSH into server and execute deployment commands
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP << 'EOF'
                            cd /home/ec2-user/app
                            unzip -o ../my_app.zip
                            source venv/bin/activate
                            pip install -r requirements.txt
                            sudo systemctl restart flask-app
                        EOF
                    '''
                }
            }
        }
    }
}
```

### Key Deployment Commands Explained

**Copy files to server (SCP):**
```bash
scp -i $SSH_KEY -o StrictHostKeyChecking=no my_app.zip $USERNAME@$SERVER_IP:/home/ec2-user/
```
- `-i`: Specifies SSH key for authentication
- `-o StrictHostKeyChecking=no`: Skips host key verification prompt
- Copies zip file to the production server's home directory

**Execute remote commands (SSH with Here Document):**
```bash
ssh -i $SSH_KEY -o StrictHostKeyChecking=no $USERNAME@$SERVER_IP << 'EOF'
    # Commands run on remote server
    unzip -o ../my_app.zip
    source venv/bin/activate
    pip install -r requirements.txt
    sudo systemctl restart flask-app
EOF
```

The `<< 'EOF'` syntax allows you to send multiple commands to the remote server as a single script.

---

## Important Considerations

**SSH Access**: Jenkins server must have network connectivity to your production server and the appropriate SSH key configured.

**Indentation in Heredocs**: When using `<< EOF`, whitespace and tabs matter. Ensure the EOF marker is at the beginning of the line with no leading spaces.

**Exclude Unnecessary Files**: When zipping your application, exclude files like `.git/` that don't need to be on the production server.

**Alternative Approach**: For cleaner Jenkins files, consider storing deployment commands in a shell script on the production server and having Jenkins simply call that script.

**User Permissions**: Ensure the ec2-user account has appropriate permissions for operations like `sudo systemctl restart flask-app`. You may need to configure sudoers.

**Error Handling**: Add error checking in your Jenkinsfile to ensure stages fail appropriately if commands don't succeed.