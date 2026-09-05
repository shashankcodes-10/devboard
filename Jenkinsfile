pipeline{
    agent any
    stages{
        stage("Code checkout"){
            steps{
                git url: "https://github.com/shashankcodes-10/devboard.git", branch: "main"
            }
        }
        stage("Install dependencies"){
            steps{
                sh '''
                    cd backend
                    go mod download

                    cd ../frontend
                    npm ci --legacy-peer-deps
                  '''
            }
        }
        stage("Unit test"){
            steps{
                 sh '''
                    cd backend
                    go test ./...

                   cd ../frontend
                   npm run test
                   '''
            }
        }
        stage("SonarQube code anaylysis"){
            steps{
                withSonarQubeEnv('sonar-server') {
                     sh """
                        ${tool 'sonar-scanner'}/bin/sonar-scanner \
                        -Dsonar.projectKey=devboard \
                        -Dsonar.projectName=devboard \
                        -Dsonar.sources=backend,frontend/src \
                        -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/*_test.go
                        """
                    }
            }    
        }
        stage("OWASP dependency check"){
            steps {
                     withCredentials([
                        string(
                           credentialsId: 'nvd-api-key',
                           variable: 'NVD_API_KEY'
                   )
           ]) {
            dependencyCheck(
                odcInstallation: 'owasp',
                additionalArguments: '--scan . --nvdApiKey $NVD_API_KEY'
            )
        }
    }
        }
        stage("Trivy filesystem scan"){
            steps{
                 sh 'trivy fs --format json --output trivy-fs-report.json .'
                 }
           post {
               always {
                     archiveArtifacts artifacts: 'trivy-fs-report.json', fingerprint: true
                 }
            }
        }
        stage("DHI login "){
            steps{
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhubcreds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_TOKEN'
            )
        ]) {
            sh '''
                echo "$DOCKER_TOKEN" | docker login dhi.io \
                    -u "$DOCKER_USERNAME" \
                    --password-stdin
            '''
                }
            }
        }
        stage("Build docker image "){
            steps{
                 sh '''
            docker build -t devboard-backend:latest ./backend
            docker build  -t devboard-frontend:latest ./frontend
            '''
            }
        }
        stage("Trivy image scan"){
            steps{
                 sh '''
            trivy image --format json --output backend-image-report.json devboard-backend:latest
            trivy image --format json --output frontend-image-report.json devboard-frontend:latest
                   '''
             }
            post {
               always {
                   archiveArtifacts artifacts: '*-image-report.json', fingerprint: true
                     }
               }
         }
        stage("Push to dockerHub"){
            steps{
                withCredentials([
                     usernamePassword(
                          credentialsId: 'dockerhubcreds',
                          usernameVariable: 'DOCKER_USERNAME',
                          passwordVariable: 'DOCKER_PASSWORD'
                      )
                 ]) {
                sh '''
                    docker tag devboard-backend:latest "$DOCKER_USERNAME/devboard-backend:latest"
                    docker tag devboard-frontend:latest "$DOCKER_USERNAME/devboard-frontend:latest"

                    echo "$DOCKER_PASSWORD" | docker login \
                        -u "$DOCKER_USERNAME" \
                        --password-stdin

                    docker push "$DOCKER_USERNAME/devboard-backend:latest"
                   docker push "$DOCKER_USERNAME/devboard-frontend:latest"

                  docker logout
              '''
                }
            }
        }
        stage("Deploy"){
            steps{
                 sh '''
                       docker compose pull
                       docker compose up -d --force-recreate
                    '''
            }
        }
        stage("Smoke testing"){
            steps{
                 sh '''
                    curl -f http://localhost:80
                    curl -f http://localhost:8081/health
                    ''' 
            }
        }
        stage("Email notification"){
            steps{
                echo " sended the notification , pls check"
            }
        }
     }
    post {
       success {
           emailext(
              subject: "Jenkins Build Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
              body: """
                Build Successful!

                Job: ${env.JOB_NAME}
                Build: #${env.BUILD_NUMBER}
                Status: ${currentBuild.currentResult}

                Check Jenkins for complete build details.
            """,
            to: "u10shashank@gmail.com"
        )
    }

    failure {
        emailext(
            subject: "Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: """
                Build Failed!

                Job: ${env.JOB_NAME}
                Build: #${env.BUILD_NUMBER}
                Status: ${currentBuild.currentResult}

                Check Jenkins console output for details.
            """,
            to: "your-email@gmail.com"
           )
         }
      }        
    }
