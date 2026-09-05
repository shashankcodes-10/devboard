# DevBoard --- Jenkins DevSecOps CI/CD Pipeline

This README documents the Jenkins setup used for the DevBoard project on
Ubuntu AWS EC2. It follows the actual Jenkinsfile configuration,
including the exact global tool names and credential IDs.

## 1. Application

-   Frontend: React / Vite / Node.js
-   Backend: Go
-   Database: PostgreSQL
-   Web server: Nginx
-   Containerization: Docker / Docker Compose
-   CI/CD: Jenkins
-   Code analysis: SonarQube
-   Dependency scan: OWASP Dependency-Check
-   Security scanning: Trivy
-   Base images: Docker Hardened Images (DHI)
-   Registry: Docker Hub
-   Notification: Jenkins Email Extension

## 2. Pipeline Flow

``` text
Code Checkout
     ↓
Install Dependencies
     ↓
Unit Tests
     ↓
SonarQube Analysis
     ↓
OWASP Dependency Check
     ↓
Trivy Filesystem Scan
     ↓
DHI Login
     ↓
Docker Image Build
     ↓
Trivy Image Scan
     ↓
Push to Docker Hub
     ↓
Deploy
     ↓
Smoke Testing
     ↓
Email Notification
```

## 3. EC2 Requirements

Run the Jenkins server and application deployment on Ubuntu EC2.

Common Security Group ports:

  Port   Purpose
  ------ ----------------------
  22     SSH
  80     DevBoard application
  8080   Jenkins
  9000   SonarQube

Do not expose PostgreSQL publicly unless required.

## 4. Ubuntu Preparation

``` bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git curl wget unzip
```

Verify Java:

``` bash
java -version
```

If required:

``` bash
sudo apt install -y fontconfig openjdk-21-jre
```

## 5. Jenkins

Install Jenkins using the current Jenkins Ubuntu installation
instructions.

``` bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

Initial administrator password:

``` bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open:

``` text
http://<EC2-PUBLIC-IP>:8080
```

Install the required plugins, including Pipeline, Git, SonarQube
Scanner, OWASP Dependency-Check and Email Extension.

## 6. Docker

Install Docker Engine and Docker Compose.

``` bash
docker --version
docker compose version
```

Allow Jenkins to use Docker:

``` bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Verify:

``` bash
docker ps
```

## 7. Go and Node.js

Backend:

``` bash
go version
```

Frontend:

``` bash
node -v
npm -v
```

## 8. Trivy

Install Trivy using the official Ubuntu installation method.

``` bash
trivy --version
```

Trivy is used for both filesystem and Docker image scans.

## 9. SonarQube

SonarQube runs on the same EC2 instance as Jenkins.

Example:

``` bash
docker run -itd --name sonarqube-server \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  -v sonarqube_logs:/opt/sonarqube/logs \
  sonarqube:community
```

Open:

``` text
http://<EC2-PUBLIC-IP>:9000
```

Optional automatic restart:

``` bash
docker update --restart unless-stopped sonarqube-server
```

## 10. SonarQube Jenkins Configuration

Create a SonarQube token and save it in Jenkins as:

``` text
Kind: Secret text
ID: sonar
```

Go to `Manage Jenkins → System → SonarQube servers` and use:

``` text
Name: sonar-server
Server URL: http://localhost:9000
Credential: sonar
```

The Jenkinsfile references the server as `sonar-server`.

Under `Manage Jenkins → Tools`, configure SonarQube Scanner:

``` text
Name: sonar-scanner
Version used: 8.1.0.6389
```

The Jenkinsfile references the scanner as `sonar-scanner`.

## 11. OWASP Dependency-Check

Under `Manage Jenkins → Tools`, configure:

``` text
Name: owasp
Version: 13.0.0
```

The Jenkinsfile references this exact name with
`odcInstallation: 'owasp'`.

## 12. NVD API Key

Create an NVD API key and store it in Jenkins as:

``` text
Kind: Secret text
ID: nvd-api-key
```

The Jenkinsfile injects the credential as the environment variable
`NVD_API_KEY`.

Never hard-code the API key in the Jenkinsfile or commit it to GitHub.

## 13. DHI Credentials

The project uses Docker Hardened Images, so Jenkins needs authentication
to `dhi.io`.

Create:

``` text
Kind: Username with password
ID: dockerhubcreds
```

The Jenkinsfile uses:

``` text
usernameVariable: DOCKER_USERNAME
passwordVariable: DOCKER_TOKEN
```

The DHI login is performed with Docker login and `--password-stdin`.

## 14. Docker Hub Credentials

The **same Jenkins credential** is used for Docker Hub:

``` text
ID: dockerhubcreds
```

Use a Docker Hub access token as the password value.

Therefore the exact credential IDs used by the current Jenkinsfile are:

  Credential ID      Purpose
  ------------------ -----------------------------
  `sonar`            SonarQube token
  `nvd-api-key`      NVD API key
  `dockerhubcreds`   DHI login + Docker Hub push

## 15. Gmail Email Configuration

Go to:

``` text
Manage Jenkins → System → Extended E-mail Notification
```

Current SMTP configuration:

``` text
SMTP server: smtp.gmail.com
SMTP port: 465
Use SSL: enabled
```

Use the Gmail credential configured in Jenkins and a Gmail App Password
where required.

The current success email recipient is:

``` text
u10shashank@gmail.com
```

The current failure section still contains `your-email@gmail.com`;
replace it with the intended recipient.

## 16. Code Checkout

The pipeline checks out the `main` branch:

``` groovy
git url: 'https://github.com/shashankcodes-10/devboard.git', branch: 'main'
```

## 17. Install Dependencies

Backend:

``` bash
cd backend
go mod download
```

Frontend:

``` bash
cd frontend
npm ci --legacy-peer-deps
```

## 18. Unit Tests

Backend:

``` bash
cd backend
go test ./...
```

Frontend:

``` bash
cd frontend
npm run test
```

## 19. SonarQube Analysis

The current Jenkinsfile analyzes:

``` text
Project key: devboard
Project name: devboard
Sources: backend,frontend/src
Exclusions: node_modules, dist, *_test.go
```

The Jenkins global names used are:

``` text
sonar-server
sonar-scanner
```

## 20. OWASP Dependency Check

The pipeline uses:

``` text
Tool: owasp
Credential: nvd-api-key
Environment variable: NVD_API_KEY
```

The NVD key is passed to Dependency-Check through the `--nvdApiKey`
option.

## 21. Trivy Filesystem Scan

The current pipeline intentionally uses Trivy in **report-only mode**:

``` bash
trivy fs --format json --output trivy-fs-report.json .
```

The report is archived as a Jenkins artifact.

Do not add `--exit-code 1` if vulnerabilities should not automatically
stop the pipeline.

## 22. DHI Login

The pipeline logs in to `dhi.io` before the Docker build using:

``` text
Credential: dockerhubcreds
Username variable: DOCKER_USERNAME
Password variable: DOCKER_TOKEN
```

Keep the credentials masked and never echo the token.

## 23. Docker Image Build

Backend:

``` bash
docker build -t devboard-backend:latest ./backend
```

Frontend:

``` bash
docker build -t devboard-frontend:latest ./frontend
```

## 24. Trivy Image Scan

The two locally built images are scanned:

``` bash
trivy image --format json --output backend-image-report.json devboard-backend:latest
trivy image --format json --output frontend-image-report.json devboard-frontend:latest
```

Both reports are archived by Jenkins.

## 25. Docker Hub Push

The images are tagged using the Jenkins Docker Hub username:

``` text
$DOCKER_USERNAME/devboard-backend:latest
$DOCKER_USERNAME/devboard-frontend:latest
```

Then they are pushed to Docker Hub using the `dockerhubcreds`
credential.

## 26. Deployment `.env`

Docker Compose requires runtime environment variables.

For this setup, the `.env` file was created manually on the EC2 server
using:

``` bash
sudo vim .env
```

The deployment configuration contains values such as:

``` env
POSTGRES_USER=admin
POSTGRES_PASSWORD=<secret>
POSTGRES_DB=devboard
POSTGRES_HOST_PORT=5432
BACKEND_HOST_PORT=8081
FRONTEND_HOST_PORT=80
```

Use the exact values required by the current `docker-compose.yml`.

### Important

Do not commit the real `.env` file to GitHub because it contains the
PostgreSQL password.

The Jenkins workspace is:

``` text
/var/lib/jenkins/workspace/devboard-devsecops
```

If `.env` is stored in the Jenkins workspace, remember that the file
permissions/owner must allow Jenkins to read it. A protected deployment
`.env` outside the workspace is preferable.

## 27. Deploy

The current Jenkinsfile uses:

``` bash
docker compose pull
docker compose up -d --force-recreate
```

`docker compose pull` gets the latest images from Docker Hub.

`docker compose up -d --force-recreate` recreates the containers so the
newly pulled images are used.

Do not run `docker compose build` in this deployment stage because the
images were already built and scanned earlier in the pipeline.

## 28. Verify Deployment

``` bash
docker compose ps
docker ps
```

Check logs:

``` bash
docker logs <container-name>
```

PostgreSQL logs:

``` bash
docker logs <postgres-container-name>
```

## 29. Smoke Testing

The current smoke test is:

``` bash
curl -f http://localhost:80
curl -f http://localhost:8081/health
```

Because Jenkins is running on the same EC2 instance as the application,
`localhost` refers to that EC2 machine.

From your laptop/browser, use the EC2 public IP instead:

``` text
http://<EC2-PUBLIC-IP>:80
```

The smoke test checks the frontend/Nginx endpoint and the Go backend
health endpoint.

PostgreSQL does not need a separate HTTP smoke test. Database readiness
is handled by the Docker/Compose healthcheck and the backend depends on
the database.

## 30. Email Notification

The Jenkinsfile has an `Email notification` stage that currently prints
a message, while the actual emails are sent by the pipeline-level `post`
block.

Success notification:

``` text
Subject: Jenkins Build Successful: <JOB_NAME> #<BUILD_NUMBER>
Recipient: u10shashank@gmail.com
```

Failure notification uses the same build information, but its current
recipient is `your-email@gmail.com` and should be changed.

### `post` placement

`post` must be outside the `stages` block:

``` groovy
pipeline {
    agent any
    stages {
        // stages
    }
    post {
        success {
            // success email
        }
        failure {
            // failure email
        }
    }
}
```

Putting `post` directly inside `stages` causes the Jenkins error:

``` text
Expected a stage
```

## 31. Troubleshooting Commands

Jenkins:

``` bash
sudo systemctl status jenkins
sudo systemctl restart jenkins
```

Docker:

``` bash
docker ps
docker ps -a
```

Compose:

``` bash
docker compose ps
docker compose config
```

`docker compose config` is useful when Compose reports missing
environment variables.

Logs:

``` bash
docker logs <container-name>
docker logs -f <container-name>
```

## 32. PostgreSQL Data Safety

Do not run this unless you intentionally want to delete the database
volume:

``` bash
docker compose down -v
```

Normal deployments should recreate containers without deleting the
PostgreSQL volume.

## 33. Security Rules

Never commit:

``` text
.env
NVD API keys
SonarQube tokens
Docker Hub access tokens
DHI credentials
Gmail passwords
Gmail App Passwords
```

Store CI/CD credentials in Jenkins Credentials and keep deployment
secrets protected.

## 34. Final Checklist

-   [ ] EC2 configured
-   [ ] Java installed
-   [ ] Jenkins running
-   [ ] Docker and Compose installed
-   [ ] Jenkins can use Docker
-   [ ] Go installed
-   [ ] Node.js/npm installed
-   [ ] Trivy installed
-   [ ] SonarQube running
-   [ ] SonarQube server name: `sonar-server`
-   [ ] SonarScanner name: `sonar-scanner`
-   [ ] OWASP tool name: `owasp`
-   [ ] Sonar credential: `sonar`
-   [ ] NVD credential: `nvd-api-key`
-   [ ] Docker credential: `dockerhubcreds`
-   [ ] Gmail SMTP configured
-   [ ] Deployment `.env` configured
-   [ ] Unit tests pass
-   [ ] SonarQube analysis works
-   [ ] OWASP scan works
-   [ ] Trivy filesystem report generated
-   [ ] Docker images build
-   [ ] Trivy image reports generated
-   [ ] Images push to Docker Hub
-   [ ] Deployment succeeds
-   [ ] Smoke tests pass
-   [ ] Email notification works

## 35. End-to-End Result

``` text
Developer pushes code
        ↓
GitHub
        ↓
Jenkins
        ↓
Checkout → Dependencies → Tests
        ↓
SonarQube → OWASP → Trivy FS
        ↓
DHI Login → Docker Build
        ↓
Trivy Image Scan
        ↓
Docker Hub Push
        ↓
Docker Compose Deploy
        ↓
Smoke Test
        ↓
Email Notification
```

This README reflects the Jenkins global names, credential IDs,
deployment approach, smoke tests, and notification setup used in the
current DevBoard pipeline.
