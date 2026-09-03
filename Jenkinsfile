pipeline{
    agent any
    stages{
        stage("cloning the code"){
            steps{
                git url: "https://github.com/shashankcodes-10/devboard.git" , branch : "main"
            }
        }
        stage('DHI Login') {
           steps {
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
        stage("building frontend image"){
            steps{
                sh "docker build -t devboard-frontend ./frontend"
            }
        }
        stage("building backend image"){
            steps{
                sh " docker build -t devboard-backend ./backend"
            }
        }
        stage('Push Image') {
        steps {
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
        stage('Deploy') {
            steps {
               withCredentials([file(credentialsId: 'devboard-env-file', variable: 'ENV_FILE')]) {
                sh '''
                   rm -f .env
                   cp "$ENV_FILE" .env
                   docker compose up -d
                  '''
                 }
             }
           }
    }
}
