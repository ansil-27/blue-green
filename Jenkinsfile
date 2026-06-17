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
                          --output text \
                          --no-cli-pager
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Active TG: ${activeTG}"

                    if (activeTG.contains("tg-blue")) {
                        env.TARGET_ENV  = "green"
                        env.TARGET_TG   = env.TG_GREEN_ARN
                        env.ROLLBACK_TG = env.TG_BLUE_ARN
                    } else {
                        env.TARGET_ENV  = "blue"
                        env.TARGET_TG   = env.TG_BLUE_ARN
                        env.ROLLBACK_TG = env.TG_GREEN_ARN
                    }

                    echo "Deploying to: ${env.TARGET_ENV}"
                }
            }
        }

        stage('Run Candidate Container') {
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

        stage('Smoke Test') {
            steps {
                sh '''
                echo "Testing candidate..."
                curl -f http://localhost:8000/health || curl -f http://localhost:8000/
                '''
            }
        }

        stage('Switch Traffic (ALB)') {
            steps {
                sh '''
                aws elbv2 modify-listener \
                  --listener-arn "$LISTENER_ARN" \
                  --default-actions Type=forward,TargetGroupArn=$TARGET_TG \
                  --no-cli-pager
                '''
            }
        }

        stage('Verify Deployment') {
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

        stage('Finalize Deployment') {
            steps {
                sh '''
                docker stop candidate-app || true
                docker rm candidate-app || true

                docker run -d \
                  --name ${TARGET_ENV}-app \
                  -p 80:80 \
                  $IMAGE
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
            docker rm -f candidate-app || true

            aws elbv2 modify-listener \
              --listener-arn "$LISTENER_ARN" \
              --default-actions Type=forward,TargetGroupArn=$ROLLBACK_TG \
              --no-cli-pager || true
            '''
            echo "Rollback Completed"
        }
    }
}