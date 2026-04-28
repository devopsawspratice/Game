pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    environment {
        DOCKER_USER = "devopsawspractice"
        IMAGE_NAME = "sliding-block-puzzle-game"
        IMAGE_TAG = "${BUILD_NUMBER}"   // dynamic tagging (better than v1)
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
        NEXUS_URL = "http://54.85.118.103:8081/repository/game/"
        RECIPIENTS = "sharan0434@gmail.com"
    }

    stages {

        // -------------------------------
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/devopsawspratice/Game.git'
            }
        }

        // -------------------------------
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        // -------------------------------
       


        // -------------------------------
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        // -------------------------------
        stage('Package Artifact') {
            steps {stage('SonarQube Analysis') {
    steps {
        script {
            def scannerHome = tool 'sq'
            withSonarQubeEnv('sq') {
                sh """
                ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=game \
                -Dsonar.sources=src \
                -Dsonar.projectName=game-App \
                -Dsonar.projectVersion=${BUILD_NUMBER}
                """
            }
        }
    }
}
                sh '''
                if [ -d dist ]; then
                    tar -czf app-${BUILD_NUMBER}.tar.gz dist
                else
                    tar -czf app-${BUILD_NUMBER}.tar.gz .
                fi
                '''
            }
        }

        // -------------------------------
        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus',
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

        // -------------------------------
        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .
                docker tag $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG $DOCKER_USER/$IMAGE_NAME:latest
                '''
            }
        }

        // -------------------------------
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                    docker push $DOCKER_USER/$IMAGE_NAME:latest
                    '''
                }
            }
        }

        // -------------------------------
        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                aws eks update-kubeconfig --region us-east-1 --name saran

                kubectl apply -f deployment.yml
                kubectl apply -f service.yml

                
                '''
            }
        }

        // -------------------------------
        stage('Install Helm') {
            steps {
                sh '''
                if [ ! -f helm ]; then
                    curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
                    tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
                    mv linux-amd64/helm ./helm
                    chmod +x ./helm
                fi
                '''
            }
        }

        // -------------------------------
        stage('Deploy Monitoring') {
            steps {
                sh '''
                ./helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                ./helm repo update

                ./helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                --namespace monitoring --create-namespace
                '''
            }
        }

        // -------------------------------
        stage('Expose Grafana') {
            steps {
                sh '''
                echo "Waiting for Grafana..."
                sleep 30

                kubectl patch svc monitoring-grafana \
                -n monitoring \
                -p '{"spec": {"type": "LoadBalancer"}}'

                kubectl get svc -n monitoring
                '''
            }
        }
    }

    // ==========================================
    post {

        success {
            emailext(
                subject: "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                Build SUCCESS 🎉

                Job: ${env.JOB_NAME}
                Build: ${env.BUILD_NUMBER}
                URL: ${env.BUILD_URL}
                """,
                to: "${env.RECIPIENTS}"
            )
        }

        failure {
            emailext(
                subject: "❌ FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                Build FAILED ❌

                Job: ${env.JOB_NAME}
                Build: ${env.BUILD_NUMBER}
                URL: ${env.BUILD_URL}
                """,
                to: "${env.RECIPIENTS}"
            )
        }

        always {
            archiveArtifacts artifacts: 'app-*.tar.gz', fingerprint: true, allowEmptyArchive: true
        }
    }
}
