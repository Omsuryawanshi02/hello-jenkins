
pipeline {
    agent any

    environment {
        // Docker Desktop executable path
        DOCKER = 'C:\\Program Files\\Docker\\Docker\\resources\\bin\\docker.exe'

        // AWS Configuration
        REGION   = 'ap-south-1'
        ACCOUNT  = '525830808827'
        REPO     = 'hello-jenkins'

        // Replace none with your actual EC2 instance ID
        INSTANCE = 'none'

        REGISTRY = "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com"
        IMG      = "${REPO}:${BUILD_NUMBER}"
        REMOTE   = "${REGISTRY}/${REPO}:latest"
    }

    stages {

        // 1. Docker Check
        stage('Docker Check') {
            steps {
                bat '''
                    if not exist "%DOCKER%" (
                        echo Docker executable not found!
                        exit /b 1
                    )

                    "%DOCKER%" --version
                    "%DOCKER%" info
                '''
            }
        }

        // 2. Build Docker Image
        stage('Build') {
            steps {
                bat '"%DOCKER%" build -t "%IMG%" .'
            }
        }

        // 3. Test Docker Image
        stage('Test') {
            steps {
                bat '"%DOCKER%" run --rm "%IMG%" nginx -t'
            }
        }

        // 4. Login to AWS ECR and Push Image
        stage('Push') {
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
                    script {
                        try {
                            bat '''
                                echo Logging in to AWS ECR...

                                "%DOCKER%" run --rm ^
                                -e AWS_ACCESS_KEY_ID ^
                                -e AWS_SECRET_ACCESS_KEY ^
                                -e AWS_DEFAULT_REGION=%REGION% ^
                                amazon/aws-cli:latest ^
                                ecr get-login-password --region %REGION% > ecr-password.txt

                                if errorlevel 1 exit /b 1

                                "%DOCKER%" login ^
                                --username AWS ^
                                --password-stdin %REGISTRY% < ecr-password.txt

                                if errorlevel 1 exit /b 1
                            '''

                            bat '''
                                echo Tagging Docker image...

                                "%DOCKER%" tag "%IMG%" "%REMOTE%"

                                if errorlevel 1 exit /b 1

                                echo Pushing image to AWS ECR...

                                "%DOCKER%" push "%REMOTE%"
                            '''
                        } finally {
                            bat '''
                                if exist ecr-password.txt del /f /q ecr-password.txt
                            '''
                        }
                    }
                }
            }
        }

        // 5. Deploy to EC2 using AWS SSM
        stage('Deploy') {
            when {
                expression {
                    return env.INSTANCE != 'none' &&
                           env.INSTANCE?.trim()
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
                        if not exist deploy.json (
                            echo deploy.json file not found!
                            exit /b 1
                        )

                        "%DOCKER%" run --rm ^
                        -e AWS_ACCESS_KEY_ID ^
                        -e AWS_SECRET_ACCESS_KEY ^
                        -e AWS_DEFAULT_REGION=%REGION% ^
                        -v "%WORKSPACE%:/workspace" ^
                        -w /workspace ^
                        amazon/aws-cli:latest ^
                        ssm send-command ^
                        --region %REGION% ^
                        --instance-ids %INSTANCE% ^
                        --document-name AWS-RunShellScript ^
                        --parameters file://deploy.json ^
                        --comment "Jenkins Docker deployment"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Console Output.'
        }

        always {
            echo 'Jenkins pipeline execution finished.'
        }
    }
}
