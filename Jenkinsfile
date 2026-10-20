
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        IMAGE_NAME = 'YOUR_DOCKERHUB_USERNAME/devops-cicd-lab'
        APP_HOST   = 'YOUR_APP_PRIVATE_IP'
        APP_USER   = 'ubuntu'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    set -eu
                    test -s index.html
                    test -s Dockerfile
                    grep -q 'Deployment Successful' index.html
                    echo "Source validation passed"
                '''
            }
        }

        stage('Build and Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x
                        trap 'docker logout >/dev/null 2>&1 || true' EXIT

                        printf '%s' "$DH_TOKEN" |
                            docker login -u "$DH_USER" --password-stdin

                        docker build --pull \
                            -t "$IMAGE_NAME:$BUILD_NUMBER" \
                            -t "$IMAGE_NAME:latest" .

                        docker push "$IMAGE_NAME:$BUILD_NUMBER"
                        docker push "$IMAGE_NAME:latest"
                    '''
                }
            }
        }

        stage('Deploy to App EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'app-ec2-deploy-key',
                        keyFileVariable: 'SSH_KEY'
                    ),
                    file(
                        credentialsId: 'app-ec2-known-hosts',
                        variable: 'KNOWN_HOSTS'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        ssh -i "$SSH_KEY" \
                            -o IdentitiesOnly=yes \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o UserKnownHostsFile="$KNOWN_HOSTS" \
                            "$APP_USER@$APP_HOST" \
                            "sudo docker pull $IMAGE_NAME:$BUILD_NUMBER &&
                             (sudo docker rm -f devops-cicd-web || true) &&
                             sudo docker run -d \
                               --name devops-cicd-web \
                               --restart unless-stopped \
                               -p 80:80 \
                               $IMAGE_NAME:$BUILD_NUMBER &&
                             sleep 3 &&
                             sudo docker exec devops-cicd-web \
                               wget -qO- http://127.0.0.1/ |
                               grep -q 'Deployment Successful' "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD deployment completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check Console Output for details.'
        }
    }
}

