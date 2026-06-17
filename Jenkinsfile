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
                sh 'docker build -t $IMAGE .'
            }
        }

        stage('Detect Active Target Group') {
            steps {
                script {

                    def activeTG = sh(
                        script: """
                        aws elbv2 describe-listeners \
                          --listener-arns "$LISTENER_ARN" \
                          --query 'Listeners[0].DefaultActions[0].TargetGroupArn' \
                          --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Active TG: ${activeTG}"

                    if (activeTG.contains("tg-blue")) {
                        env.TARGET_ENV  = "green"
                        env.TARGET_PORT = "8001"
                        env.TARGET_TG   = env.TG_GREEN_ARN
                        env.ROLLBACK_TG = env.TG_BLUE_ARN
                    } else {
                        env.TARGET_ENV  = "blue"
                        env.TARGET_PORT = "8000"
                        env.TARGET_TG   = env.TG_BLUE_ARN
                        env.ROLLBACK_TG = env.TG_GREEN_ARN
                    }

                    echo "Deploying to: ${env.TARGET_ENV}"
                }
            }
        }

        stage('Deploy New Version') {
            steps {
                sh '''
                docker stop ${TARGET_ENV}-app || true
                docker rm ${TARGET_ENV}-app || true

                docker run -d \
                  --name ${TARGET_ENV}-app \
                  -p ${TARGET_PORT}:80 \
                  $IMAGE
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                curl -f http://localhost:${TARGET_PORT}/
                '''
            }
        }

        stage('Switch Traffic (ALB)') {
            steps {
                sh '''
                aws elbv2 modify-listener \
                  --listener-arn "$LISTENER_ARN" \
                  --default-actions Type=forward,TargetGroupArn=$TARGET_TG
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                sleep 10
                aws elbv2 describe-listeners \
                  --listener-arns "$LISTENER_ARN" \
                  --query 'Listeners[0].DefaultActions[0].TargetGroupArn' \
                  --output text
                '''
            }
        }
    }

    post {

        success {
            echo "Blue-Green Deployment SUCCESS"
        }

        failure {
            sh '''
            aws elbv2 modify-listener \
              --listener-arn "$LISTENER_ARN" \
              --default-actions Type=forward,TargetGroupArn=$ROLLBACK_TG || true
            '''
            echo "Rollback Completed"
        }
    }
}