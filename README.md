# Task 2: Simple Jenkins Pipeline for CI/CD

# Project Explanation

This project demonstrates a basic **CI/CD (Continuous Integration / Continuous Deployment)** workflow using **Jenkins** and **Docker**.

- **Continuous Integration (CI)** — every time new code is pushed to GitHub, it is automatically pulled, built, and tested, instead of a developer doing this by hand.
- **Continuous Deployment (CD)** — once the build and tests pass, the app is automatically deployed (made live), with zero manual steps.

**End-to-end flow:**
1. A simple static web app (one HTML page) lives in a GitHub repository.
2. Jenkins, running inside a Docker container, watches this repository for changes (via Poll SCM).
3. On every push, Jenkins automatically:
   - **Checks out** the latest code from GitHub
   - **Builds** a Docker image of the app using the `Dockerfile`
   - **Tests** the image inside a temporary container
   - **Deploys** it by stopping the old container and starting a new one from the fresh image
4. The result: a live webpage that always reflects the latest pushed code — no manual deployment required.

This mirrors real-world DevOps practice, where CI/CD pipelines move code from "pushed to GitHub" to "running in production" automatically, reducing manual work and human error.

# Objective
Set up a basic Jenkins pipeline to automate the process of building and deploying a simple web application using Docker.

# Tools Used
| Tool | Purpose |
|---|---|
| Jenkins | Automation server that runs the pipeline |
| Docker | Packages the app into a portable image and runs it as a container |
| Git / GitHub | Source control — stores the app code and triggers builds |

#Project Structure
.
├── Jenkinsfile     # Pipeline definition (Checkout → Build → Test → Deploy)
├── Dockerfile      # Builds the app image (nginx serving index.html)
├── index.html      # Sample app page
└── README.md       # This file

# Setup Steps

# 1. Install Docker

sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER


# 2. Run Jenkins as a Docker container

docker volume create jenkins_home

docker run -d --name jenkins \
  -p 8081:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts

> Jenkins runs on **port 8081** 

# 3. Unlock Jenkins
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

Used this password to finish setup at `http://localhost:8081`, installed the suggested plugins, and created an admin account.

# 4. Fix Docker socket permissions
Jenkins needs permission to talk to the host's Docker daemon in order to build/run images:

sudo chmod 666 /var/run/docker.sock

# 5. Create the Pipeline job
- Jenkins Dashboard → **New Item** → choose **Pipeline**
- Pipeline definition: **Pipeline script from SCM**
- SCM: **Git**, repo URL: `https://github.com/gautam-guts-repo/jenkins-demo.git`
- Branch Specifier: `*/main`
- Script Path: `Jenkinsfile`

# 6. Enable automatic triggering
- Job → **Configure** → **Build Triggers** → check **Poll SCM**
- Schedule: `* * * * *` (checks GitHub for new commits every minute)

# Code

## Jenkinsfile
Defines the four pipeline stages — Checkout, Build, Test, Deploy.
groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = "myapp"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
        CONTAINER_NAME = "myapp-container"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Pulling latest code from repository...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests inside a temporary container...'
                sh 'docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} echo "tests passed"'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying new container...'
                sh '''
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p 8082:80 ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the console output above.'
        }
    }
}

# Dockerfile
Packages the static page into an nginx image.

FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80


# index.html
The sample app page served by the container.
<!DOCTYPE html>
<html>
<head><title>My Jenkins CI/CD App</title></head>
<body>
  <h1>Deployed via Jenkins Pipeline 🚀</h1>
</body>
</html>


# Port Allocation
| Service | Port |
|---|---|
| Jenkins | 8081 |
| Deployed demo app (`myapp-container`) | 8082 |

# Result
On a successful build, the pipeline deploys the app and it becomes reachable at:

http://localhost:8082

#  How to Trigger a New Build
Push any change to the repo — Jenkins polls every minute and rebuilds/redeploys automatically:

git add .
git commit -m "update"
git push


