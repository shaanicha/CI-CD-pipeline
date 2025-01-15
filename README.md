# CI-CD-pipeline

# CI/CD Pipeline Setup

This project automates the continuous integration (CI) and continuous deployment (CD) process using **Jenkins**. It covers the entire lifecycle from code commit to deployment on the staging environment using Docker containers.

## Project Structure

The project follows the standard structure for a Python-based application with Docker integration. Below is the structure of the repository:

project-root/ ├── app/ │ ├── init.py │ ├── main.py │ ├── ... (other app files) ├── tests/ │ ├── test_main.py │ ├── ... (other test files) ├── requirements.txt ├── Dockerfile ├── Jenkinsfile ├── README.md


## Prerequisites

Before starting with the pipeline setup, ensure you have the following:

1. **Jenkins** server set up and running.
2. **GitHub** repository with the project code.
3. **Docker** installed on the Jenkins server.
4. **Docker Hub** account to push the Docker images.
5. **GitHub Personal Access Token (PAT)** for Jenkins authentication.
6. **Email server configuration** for sending notifications.

## Jenkins Setup

### 1. Install Jenkins and Required Plugins

- Install **Jenkins** on your server (or EC2 instance).
- Install the following Jenkins plugins:
  - **Docker** plugin
  - **Git** plugin
  - **Email Extension** plugin (for email notifications)
  - **Pipeline** plugin

### 2. Create Jenkins Credentials

- **GitHub Credentials**: Add GitHub credentials in Jenkins to access the repository.
- **Docker Hub Credentials**: Add Docker Hub credentials to allow Jenkins to push images to your Docker Hub account.

### 3. Create Jenkins Pipeline Job

1. Go to the Jenkins dashboard.
2. Create a new **Pipeline** job.
3. In the **Pipeline** section, select **Pipeline script from SCM**.
4. Set the following values:
   - **SCM**: Git
   - **Repository URL**: `https://github.com/shaanicha/CI-CD-pipeline.git`
   - **Branch**: `main`
5. In the **Pipeline** script field, point to the `Jenkinsfile` in the root directory of your project.

### 4. Configure Email Notifications

- Go to **Manage Jenkins** > **Configure System** > **Extended E-mail Notification**.
- Provide SMTP server details, email sender address, and recipients to send notifications.

## Jenkins Pipeline Workflow

### Overview

The CI/CD pipeline is defined in the `Jenkinsfile` and contains the following stages:

1. **Checkout Code**: 
   - Pulls the latest code from the GitHub repository using the GitHub credentials.

2. **Install Dependencies**:
   - Creates a Python virtual environment.
   - Installs the dependencies listed in `requirements.txt`.

3. **Run Unit Tests**:
   - Runs the unit tests using **pytest** to ensure the application works as expected.

4. **Build Docker Image**:
   - Builds a Docker image using the provided `Dockerfile` and tags it with the Docker Hub username and image name.

5. **Push Docker Image**:
   - Pushes the built Docker image to **Docker Hub**.

6. **Deploy to Staging**:
   - Deploys the Docker container to a staging environment (e.g., EC2 instance) and maps the necessary ports.

### Example of a Failed Build

If any of the above stages fail (e.g., unit tests fail, Docker build fails), the pipeline will send an email notification to the specified recipients.

### Post Build Actions

- **Email Notification**: 
   - After the pipeline completes, an email is sent with details of the build status (success or failure).
   - The email contains information about the job, build number, and a link to the build details.

## Jenkinsfile

The `Jenkinsfile` defines the pipeline and consists of the following sections:

```groovy
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'shaani/ci-cd-pipeline-image'
        STAGING_ENV = 'staging-container'
    }
    stages {
        stage('Checkout Code') {
            steps {
                echo 'Checking out code from GitHub...'
                git credentialsId: 'github-credentials', branch: 'main', url: 'https://github.com/shaanicha/CI-CD-pipeline.git'
            }
        }
        stage('Install Dependencies') {
            steps {
                echo 'Installing Python dependencies...'
                script {
                    sh 'python3 -m venv venv'
                    sh '. venv/bin/activate && pip install -r requirements.txt'
                }
            }
        }
        stage('Run Unit Tests') {
            steps {
                echo 'Running unit tests...'
                script {
                    sh '. venv/bin/activate && pytest tests/'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t $DOCKER_IMAGE .'
            }
        }
        stage('Push Docker Image') {
            steps {
                echo 'Pushing Docker image to Docker Hub...'
                withDockerRegistry([credentialsId: 'docker-credentials', url: '']) {
                    sh 'docker push $DOCKER_IMAGE'
                }
            }
        }
        stage('Deploy to Staging') {
            steps {
                echo 'Deploying Docker container to staging environment...'
                sh """
                docker stop $STAGING_ENV || true
                docker rm $STAGING_ENV || true
                docker run -d --name $STAGING_ENV -p 5000:5000 $DOCKER_IMAGE
                """
            }
        }
    }
    post {
        always {
            echo 'Pipeline execution completed. Sending email notification...'
            emailext(
                subject: "Jenkins Pipeline Status: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <p>The Jenkins pipeline has completed execution.</p>
                    <p>Details:</p>
                    <ul>
                        <li><b>Status:</b> ${currentBuild.currentResult}</li>
                        <li><b>Job:</b> ${env.JOB_NAME}</li>
                        <li><b>Build Number:</b> ${env.BUILD_NUMBER}</li>
                        <li><b>URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></li>
                    </ul>
                """,
                recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                to: 'your-recipient@example.com',
                mimeType: 'text/html'
            )
        }
    }
}
