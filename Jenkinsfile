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
                to: 'shraddhachaudhari1730@gmail.com',
                mimeType: 'text/html'
            )
        }
    }
}
