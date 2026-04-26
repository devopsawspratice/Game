pipeline {
    agent any

    tools {
        nodejs 'NodeJS18'
    }

    environment {
        DOCKER_USER = "surya8442"
        IMAGE_NAME = "sliding-block-puzzle-game"
        IMAGE_TAG = "v1"
        KUBECONFIG = '/var/lib/jenkins/.kube/config'

        NEXUS_URL = "http://65.0.86.137:8081/repository/puzzlegame"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Surya8442/Game.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sq') {
                    sh '''
                        /opt/sonar-scanner/bin/sonar-scanner \
                        -Dsonar.projectKey=game \
                        -Dsonar.sources=src \
                        -Dsonar.projectName=game-App \
                        -Dsonar.projectVersion=${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                    if [ -d dist ]; then
                        tar -czf app-${BUILD_NUMBER}.tar.gz dist
                    else
                        tar -czf app-${BUILD_NUMBER}.tar.gz .
                    fi
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        curl -v -u $NEXUS_USER:$NEXUS_PASS \
                        --upload-file app-${BUILD_NUMBER}.tar.gz \
                        $NEXUS_URL/app-${BUILD_NUMBER}.tar.gz
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'Docker_cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    aws eks update-kubeconfig --region ap-south-1 --name mycluster
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml
                '''
            }
        }
    }

    post {
        success {
            echo "SUCCESS: Pipeline completed successfully"
        }

        failure {
            echo "FAILED: Check Jenkins logs for errors"
        }

        always {
            echo "Pipeline execution completed"
        }
    }
}
