pipeline {
    agent any

    environment {
        IMAGE = "bluegreen:${BUILD_NUMBER}"

        LISTENER_ARN = credentials('LISTENER_ARN')
        TG_BLUE_ARN  = credentials('TG_BLUE_ARN')
        TG_GREEN_ARN = credentials('TG_GREEN_ARN')
    }

    stages {

        stage('Build') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }

        stage('Deploy Green') {
            steps {
                sh '''
                docker rm -f green-app || true

                docker run -d \
                  --name green-app \
                  -p 8001:80 \
                  $IMAGE
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'curl -f http://localhost:8001'
            }
        }

        stage('Switch to Green') {
            steps {
                sh '''
                aws elbv2 modify-listener \
                  --listener-arn "$LISTENER_ARN" \
                  --default-actions Type=forward,TargetGroupArn=$TG_GREEN_ARN
                '''
            }
        }

        stage('Cleanup Blue') {
            steps {
                sh '''
                docker rm -f blue-app || true
                '''
            }
        }
    }

    post {
        failure {
            sh '''
            aws elbv2 modify-listener \
              --listener-arn "$LISTENER_ARN" \
              --default-actions Type=forward,TargetGroupArn=$TG_BLUE_ARN || true
            '''
            echo "Rollback to Blue"
        }

        success {
            echo "Deployment Successful"
        }
    }
}