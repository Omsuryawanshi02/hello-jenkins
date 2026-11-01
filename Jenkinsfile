
pipeline {

    agent any

    environment {
        REGION       = 'eu-north-1'
        ACCOUNT      = '525830808827'
        ECR_REPOSITORY = 'jenkins-web'

        REGISTRY = '525830808827.dkr.ecr.ap-south-1.amazonaws.com'

        IMAGE_NAME = 'hello-jenkins'

        DOCKER_EXE = 'C:\\Users\\om\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

        // Replace with your actual EC2 instance ID
       INSTANCE_ID = 'none'
    }

    stages {

        // 1. Docker Check
        stage('Docker Check') {
            steps {
                bat "\"${env.DOCKER_EXE}\" --version"
                bat "\"${env.DOCKER_EXE}\" info"
            }
        }

        // 2. Build Docker Image
        stage('Build') {
            steps {
                bat """
                    "${env.DOCKER_EXE}" build -t ${env.IMAGE_NAME}:${env.BUILD_NUMBER} .
                """
            }
        }

        // 3. Test Docker Image
        stage('Test') {
            steps {
                bat """
                    "${env.DOCKER_EXE}" run --rm ${env.IMAGE_NAME}:${env.BUILD_NUMBER} nginx -t
                """
            }
        }

        // 4. AWS Authentication
        stage('AWS Authentication') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'aws-key-id',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    bat '''
                        aws sts get-caller-identity
                    '''

                    echo 'AWS authentication successful!'
                }
            }
        }

        // 5. Push to AWS ECR
        stage('Push to AWS ECR') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'aws-key-id',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    bat """
                        aws ecr get-login-password --region ${env.REGION} | "${env.DOCKER_EXE}" login --username AWS --password-stdin ${env.REGISTRY}

                        "${env.DOCKER_EXE}" tag ${env.IMAGE_NAME}:${env.BUILD_NUMBER} ${env.REGISTRY}/${env.ECR_REPOSITORY}:${env.BUILD_NUMBER}

                        "${env.DOCKER_EXE}" tag ${env.IMAGE_NAME}:${env.BUILD_NUMBER} ${env.REGISTRY}/${env.ECR_REPOSITORY}:latest

                        "${env.DOCKER_EXE}" push ${env.REGISTRY}/${env.ECR_REPOSITORY}:${env.BUILD_NUMBER}

                        "${env.DOCKER_EXE}" push ${env.REGISTRY}/${env.ECR_REPOSITORY}:latest
                    """

                    echo 'Successfully pushed images to AWS ECR!'
                }
            }
        }

        // 6. Deploy to EC2 using SSM
        stage('Deploy to EC2') {

            when {
                expression {
                    return env.INSTANCE_ID?.startsWith('i-')
                }
            }

            steps {
                withCredentials([
                    string(
                        credentialsId: 'aws-key-id',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    bat '''
                        aws ssm send-command ^
                        --region %REGION% ^
                        --instance-ids %INSTANCE_ID% ^
                        --document-name AWS-RunShellScript ^
                        --parameters file://deploy.json ^
                        --comment "Deploy hello-jenkins application"
                    '''

                    echo 'EC2 deployment command submitted!'
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Jenkins CI/CD pipeline completed!'
        }

        failure {
            echo 'FAILED: Check Jenkins Console Output.'
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
