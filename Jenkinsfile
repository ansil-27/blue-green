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
                sh '''
                docker build -t $IMAGE .
                '''
            }
        }

        stage('Determine Active Environment') {
            steps {
                script {

                    def activeTG = sh(
                        script: '''
                        aws elbv2 describe-listeners \
                          --listener-arns "$LISTENER_ARN" \
                          --query 'Listeners[0].DefaultActions[0].TargetGroupArn' \
                          --output text \
                          --no-cli-pager
                        ''',
                        returnStdout: true
                    ).trim()

                    if (activeTG == env.TG_BLUE_ARN) {

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

                    echo "Active TG    : ${activeTG}"
                    echo "Deploying to : ${env.TARGET_ENV}"
                }
            }
        }

        stage('Deploy Candidate') {
            steps {
                sh '''
                docker rm -f candidate-app || true

                docker run -d \
                  --name candidate-app \
                  -p 8080:80 \
                  $IMAGE
                '''
            }
        }

        stage('Wait') {
            steps {
                sh 'sleep 10'
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                curl -f http://localhost:8080/
                '''
            }
        }

        stage('Promote Candidate') {
            steps {
                sh '''
                docker stop ${TARGET_ENV}-app || true
                docker rm ${TARGET_ENV}-app || true

                docker stop candidate-app

                docker commit candidate-app ${IMAGE}-promoted

                docker rm candidate-app

                docker run -d \
                  --name ${TARGET_ENV}-app \
                  -p ${TARGET_PORT}:80 \
                  ${IMAGE}-promoted
                '''
            }
        }

        stage('Switch Traffic') {
            steps {
                sh '''
                aws elbv2 modify-listener \
                  --listener-arn "$LISTENER_ARN" \
                  --default-actions Type=forward,TargetGroupArn=$TARGET_TG \
                  --no-cli-pager
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
                  --output text \
                  --no-cli-pager
                '''
            }
        }
    }

    post {

        success {
            sh '''
            docker rm -f candidate-app || true
            '''
            echo 'Blue-Green Deployment Successful'
        }

        failure {
            sh '''
            docker rm -f candidate-app || true

            aws elbv2 modify-listener \
              --listener-arn "$LISTENER_ARN" \
              --default-actions Type=forward,TargetGroupArn=$ROLLBACK_TG \
              --no-cli-pager || true
            '''
            echo 'Rollback Completed'
        }
    }
}