pipeline {
    agent any

 
    environment {
        DOCKER_USER = "surya8442"
        IMAGE_NAME = "sliding-block-puzzle-game"
        IMAGE_TAG = "v1"
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
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

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker_cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
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
                    kubectl get nodes
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml
                '''
            }
        }

    }   

    post {
        always {
            echo 'Pipeline finished.'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
