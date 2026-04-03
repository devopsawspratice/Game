pipeline {
    agent any

    tools {
        nodejs 'NodeJS18'
        jdk 'JDK17'
    }

    environment {
        DOCKER_USER = "surya8442"
        IMAGE_NAME = "sliding-block-puzzle-game"
        IMAGE_TAG = "v1"
        SONARQUBE_ENV = 'sonar-server'
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

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                        # Install Sonar Scanner if not already installed
                        npm install -g sonar-scanner

                        # Run Sonar Scanner
                        sonar-scanner \
                            -Dsonar.projectKey=puzzlegame \
                            -Dsonar.sources=src \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.login=$SONAR_AUTH_TOKEN
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
                    export KUBECONFIG=/var/lib/jenkins/.kube/config

                    # Update kubeconfig for EKS cluster
                    aws eks update-kubeconfig --region ap-south-1 --name mycluster

                    # Check nodes
                    kubectl get nodes

                    # Deploy application
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml
                '''
            }
        }
    }
    
    stage('Deploy Monitoring Stack') {
    steps {
        export KUBECONFIG=/var/lib/jenkins/.kube/config {
            sh '''
            kubectl apply -f prometheus.yml
            kubectl apply -f grafana.yml
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
