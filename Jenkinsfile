pipeline {

    agent any

    tools {
        jdk 'JDK'
        maven 'Maven'
    }

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO = '273263084701.dkr.ecr.us-east-1.amazonaws.com/canary-deploy'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
 	   steps {
               withSonarQubeEnv('SonarQube') {
                   sh '''
                       mvn sonar:sonar \
                       -Dsonar.projectKey=jenkins-end-to-end
                   '''
               }
           }
       }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${ECR_REPO}:${IMAGE_TAG} .'
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} |
                    docker login --username AWS --password-stdin ${ECR_REPO}
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh 'docker push ${ECR_REPO}:${IMAGE_TAG}'
            }
        }
       stage('Deploy to EKS') {
    	   steps {
               sh '''
                   aws eks update-kubeconfig \
                   --region ${AWS_REGION} \
                   --name helm-cluster

                   sed "s|IMAGE_PLACEHOLDER|${ECR_REPO}:${IMAGE_TAG}|g" \
                   deployment.yaml > deployment-final.yaml

                   kubectl apply -f deployment-final.yaml
                   kubectl apply -f service.yaml

                  kubectl rollout status deployment/jenkins-demo
                 '''
              }
        }
    }

    post {
        success {
            echo 'Build and ECR push completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the console output.'
        }
    }
}
