pipeline {
    agent any

    environment {
        IMAGE = "bluegreen:${BUILD_NUMBER}"

        LISTENER_ARN = credentials('LISTENER_ARN')
        TG_BLUE_ARN  = credentials('TG_BLUE_ARN')
        TG_GREEN_ARN = credentials('TG_GREEN_ARN')
    }

    stages {

        stage('Build Image') {
            steps {
                sh '''
                docker build -t $IMAGE .
                '''
            }
        }

        stage('Deploy Green') {
            steps {
                sh '''
                docker stop green-app || true
                docker rm green-app || true

                docker run -d \
                  --name green-app \
                  -p 8001:80 \
                  $IMAGE
                '''
            }
        }

        stage('Wait for Startup') {
            steps {
                sh 'sleep 15'
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                curl -f http://localhost:8001/bhvy
                '''
            }
        }

        stage('Switch Traffic') {
            steps {
                sh '''
                aws elbv2 modify-listener \
                  --listener-arn $LISTENER_ARN \
                  --default-actions Type=forward,TargetGroupArn=$TG_GREEN_ARN
                '''
            }
        }

        stage('Verify Production') {
            steps {
                sh 'sleep 10'
            }
        }

        stage('Cleanup Old Blue') {
            steps {
                sh '''
                docker stop blue-app || true
                docker rm blue-app || true

                docker rename green-app blue-app || true
                '''
            }
        }
    }

    post {

        success {
            echo 'Blue-Green Deployment Successful'
        }

        failure {
            sh '''
            docker stop green-app || true
            docker rm green-app || true

            aws elbv2 modify-listener \
              --listener-arn $LISTENER_ARN \
              --default-actions Type=forward,TargetGroupArn=$TG_BLUE_ARN || true
            '''
            echo 'Rollback Completed'
        }
    }
}