
pipeline {
    agent any

    environment {
        // Actual Docker path on your Windows machine
        DOCKER_EXE = 'C:\\Users\\om\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

        // AWS Configuration
        REGION   = 'ap-south-1'
        ACCOUNT  = '525830808827'
        REPO     = 'hello-jenkins'

        // Replace none with your actual EC2 instance ID
        INSTANCE = 'none'

        REGISTRY    = "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com"
        IMG         = "${REPO}:${BUILD_NUMBER}"
        ECR_IMAGE   = "${REGISTRY}/${REPO}:${BUILD_NUMBER}"
        LATEST_IMAGE = "${REGISTRY}/${REPO}:latest"
    }

    stages {

        // 1. Docker Check
        stage('Docker Check') {
            steps {
                bat '''
                    @echo off

                    if not exist "%DOCKER_EXE%" (
                        echo ERROR: Docker executable not found!
                        exit /b 1
                    )

                    echo Checking Docker version...
                    "%DOCKER_EXE%" --version

                    echo Checking Docker Engine...
                    "%DOCKER_EXE%" info

                    if errorlevel 1 exit /b 1
                '''
            }
        }

        // 2. Build Docker Image
        stage('Build') {
            steps {
                echo 'Building Docker image...'

                bat '''
                    "%DOCKER_EXE%" build -t "%IMG%" .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        // 3. Test Docker Image
        stage('Test') {
            steps {
                echo 'Testing Docker image...'

                bat '''
                    "%DOCKER_EXE%" image inspect "%IMG%" >NUL
                    if errorlevel 1 exit /b 1

                    echo Docker image test passed!
                '''
            }
        }

        // 4. AWS ECR Login and Push
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
                    bat '''
                        @echo off

                        echo Logging in to AWS ECR...

                        "%DOCKER_EXE%" run --rm ^
                        -e AWS_ACCESS_KEY_ID ^
                        -e AWS_SECRET_ACCESS_KEY ^
                        -e AWS_DEFAULT_REGION=%REGION% ^
                        amazon/aws-cli:latest ^
                        ecr get-login-password --region %REGION% ^
                        | "%DOCKER_EXE%" login --username AWS --password-stdin %REGISTRY%

                        if errorlevel 1 exit /b 1

                        echo Creating ECR image tags...

                        "%DOCKER_EXE%" tag "%IMG%" "%ECR_IMAGE%"
                        if errorlevel 1 exit /b 1

                        "%DOCKER_EXE%" tag "%IMG%" "%LATEST_IMAGE%"
                        if errorlevel 1 exit /b 1

                        echo Pushing versioned image...

                        "%DOCKER_EXE%" push "%ECR_IMAGE%"
                        if errorlevel 1 exit /b 1

                        echo Pushing latest image...

                        "%DOCKER_EXE%" push "%LATEST_IMAGE%"
                        if errorlevel 1 exit /b 1

                        echo ECR Push completed!
                    '''
                }
            }
        }

        // 5. Deploy to EC2 using AWS SSM
        stage('Deploy') {

            when {
                expression {
                    return env.INSTANCE?.trim() &&
                           env.INSTANCE != 'none'
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
                        @echo off

                        if not exist deploy.json (
                            echo ERROR: deploy.json not found!
                            exit /b 1
                        )

                        echo Sending deployment command to EC2...

                        "%DOCKER_EXE%" run --rm ^
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
                        --comment "Jenkins Docker Deployment"

                        if errorlevel 1 exit /b 1

                        echo Deployment command submitted!
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Jenkins pipeline completed!'
        }

        failure {
            echo 'FAILURE: Check Console Output for errors.'
        }

        always {
            echo 'Jenkins pipeline execution finished.'
        }
    }
}
