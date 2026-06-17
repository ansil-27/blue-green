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

        stage('Determine Active Environment') {
            steps {
                script {
                    ACTIVE_TG = sh(
                        script: """
                        aws elbv2 describe-listeners \
                        --listener-arns $LISTENER_ARN \
                        --query 'Listeners[0].DefaultActions[0].TargetGroupArn' \
                        --output text
                        """,
                        returnStdout: true
                    ).trim()

                    if (ACTIVE_TG == TG_BLUE_ARN) {
                        TARGET_ENV = "green"
                        TARGET_PORT = "8001"
                        TARGET_TG = TG_GREEN_ARN
                        ROLLBACK_TG = TG_BLUE_ARN
                    } else {
                        TARGET_ENV = "blue"
                        TARGET_PORT = "8000"
                        TARGET_TG = TG_BLUE_ARN
                        ROLLBACK_TG = TG_GREEN_ARN
                    }

                    env.TARGET_ENV = TARGET_ENV
                    env.TARGET_PORT = TARGET_PORT
                    env.TARGET_TG = TARGET_TG
                    env.ROLLBACK_TG = ROLLBACK_TG

                    echo "Deploying to ${TARGET_ENV}"
                }
            }
        }

        stage('Deploy') {
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

        stage('Wait') {
            steps {
                sh 'sleep 15'
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                curl -f http://localhost:$8000/
                '''
            }
        }

        stage('Switch Traffic') {
            steps {
                sh '''
                aws elbv2 modify-listener \
                  --listener-arn $LISTENER_ARN \
                  --default-actions Type=forward,TargetGroupArn=$TARGET_TG
                '''
            }
        }
    }

    post {

        success {
            echo 'Blue-Green deployment successful'
        }

        failure {
            sh '''
            aws elbv2 modify-listener \
              --listener-arn $LISTENER_ARN \
              --default-actions Type=forward,TargetGroupArn=$ROLLBACK_TG || true
            '''
            echo 'Rollback completed'
        }
    }
}