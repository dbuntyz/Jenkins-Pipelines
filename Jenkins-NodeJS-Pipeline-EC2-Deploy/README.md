
## Jenkins Installation and React.js Deployment to EC2

### Step 1: Install OpenJDK

Update your system packages and install OpenJDK 21, which is required to run Jenkins:

`sudo apt update && sudo apt upgrade -y && sudo apt install openjdk-21-jdk -y` 

### Step 2: Install Jenkins

1.  **Add Jenkins Repository and Key**:
    
    Follow the instructions on the official [Jenkins installation guide](https://www.jenkins.io/doc/book/installing/linux/) or use the commands below:
    
    bash
    
    Copy code
    
    `sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
      https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
    echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null` 
    
2.  **Install Jenkins**:
    
    `sudo apt-get update && sudo apt-get install jenkins -y` 
    
3.  **Start and Enable Jenkins**: 

    `sudo systemctl enable jenkins && sudo systemctl start jenkins` 
    
5.  **Access Initial Admin Password**:
    
    -   To complete the Jenkins setup, retrieve the initial admin password:
    
    `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` 
    

### Step 3: Configure EC2 Instance

1.  **Launch EC2 Instance**:
    
    -   Launch an EC2 instance with an appropriate AMI (e.g., Ubuntu 20.04) and configure security groups to allow SSH (port 22) and HTTP/HTTPS (ports 80/443).
2.  **Install Required Software on EC2**:
    
    -   SSH into your EC2 instance and install Apache (or Nginx) to serve the built React app.
    
    `sudo apt update && sudo apt upgrade -y && sudo apt install apache2 -y` 
    
3.  **Configure Apache for Deployment**:
    
    -   Update the Apache configuration to serve the React application from `/var/www/html`.
    
    `sudo rm -rf /var/www/html/* && sudo chown -R $USER:$USER /var/www/html` 
    
### Step 4: Install Necessary Jenkins Plugins

After setting up Jenkins, install the following plugins:

1.  **GitHub Integration Plugin**: For GitHub repository integration.
2.  **NodeJS Plugin**: To set up the Node.js environment within Jenkins.
    -   Go to **Manage Jenkins > Global Tool Configuration > NodeJS installations**.
    -   Add a new installation:
        -   **Name**: NodeJS-Tool
        -   **Version**: Latest available.
3.  **Publish Over SSH Plugin**: To deploy code to your EC2 instance.
    -   Go to **Manage Jenkins > System Configuration > Publish over SSH**.
    -   Configure the SSH connection:
        -   **Key**: Your PEM Key.
        -   **SSH Servers**:
            -   **Name**: EC2
            -   **Hostname**: [Your EC2 hostname]
            -   **Username**: [Your EC2 username]
            -   **Remote Directory**: `/`

### Step 5: Jenkins Pipeline Script for Node.js Deployment

Below is a sample Jenkins Pipeline script for checking out code, building a Node.js application, and deploying it to an EC2 instance using SSH.

    pipeline {
        agent any
    
        environment {
            // Use the configured NodeJS tool from Jenkins
            NODEJS_HOME = tool 'NodeJS-Tool'  // Replace with the exact name you set in Jenkins
            PATH = "${NODEJS_HOME}/bin:${env.PATH}"
        }
    
        stages {
            stage('Checkout Code') {
                steps {
                    // Checkout the code from your GitHub repository
                    git branch: 'main', url: 'https://github.com/dbuntyz/webapp.git'
                }
            }
    
            stage('Install Dependencies') {
                steps {
                    // Install Node.js dependencies
                    sh 'npm install'
                }
            }
    
            stage('Build Application') {
                steps {
                    // Build the application
                    sh 'npm run build'
                }
            }
    
            stage('Deploy to EC2') {
                steps {
                    sshPublisher(publishers: [
                        sshPublisherDesc(
                            configName: 'EC2', // Name of the SSH server configuration
                            transfers: [
                                sshTransfer(
                                    sourceFiles: 'build/**/*', // Files to transfer
                                    remoteDirectory: '/var/www/html', // Remote directory on EC2
                                    removePrefix: 'build', // Remove prefix from the source files
                                    remoteDirectorySDF: false,
                                    flatten: false // Do not flatten directory structure
                                )
                            ],
                            usePromotionTimestamp: false,
                            verbose: true
                        )
                    ])
                }
            }
        }
    
        post {
            success {
                echo 'Deployment successful!'
            }
            failure {
                echo 'Pipeline failed. Check the logs for more details.'
            }
        }
    }

### Step 6: Configure Security Groups and Permissions

1.  **Security Groups**:
    
    -   Ensure the EC2 security group allows incoming traffic on ports 80 (HTTP) and 22 (SSH).
2.  **SSH Key Permissions**:
    
    -   Ensure your PEM key has the correct permissions (`chmod 400 your-key.pem`) before using it for SSH connections.

### Step 7: Finalize Jenkins Configuration

1.  **Run the Pipeline**:
    
    -   Go to your Jenkins dashboard and create a new pipeline project.
        
    -   Copy the above pipeline script into the Jenkinsfile or configure it directly in the pipeline project.
        
    -   Click **Build Now** to run the pipeline.
        
    -   **Granular Steps**:
        
        -   Navigate to Jenkins Dashboard → **New Item**.
        -   Enter an item name and select **Pipeline**.
        -   Click **OK**.
        -   Scroll down to the **Pipeline** section and choose **Pipeline Script**.
        -   Paste the pipeline script and click **Save**.
        -   To execute, click **Build Now** from the left menu.
        -   Monitor the pipeline stages in **Console Output**.