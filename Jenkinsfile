
pipeline {

    agent any

    environment {
        REGION       = 'eu-north-1'
        ACCOUNT      = '525830808827'
        REPO         = 'hello-jenkins'
        INSTANCE     = 'none'

        DOCKER_EXE   = 'C:\\Users\\om\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

        REGISTRY     = "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com"
        IMG          = "hello-jenkins:${BUILD_NUMBER}"
        REMOTE       = "${REGISTRY}/${REPO}:latest"
        VERSIONED    = "${REGISTRY}/${REPO}:${BUILD_NUMBER}"
    }

    stages {

        // 1. Docker Check
        stage('Docker Check') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'docker --version'
                        sh 'docker info'
                    } else {
                        bat "\"${env.DOCKER_EXE}\" --version"
                        bat "\"${env.DOCKER_EXE}\" info"
                    }
                }
            }
        }

        // 2. Build Docker Image
        stage('Build') {
            steps {
                script {
                    if (isUnix()) {
                        sh "docker build -t ${env.IMG} ."
                    } else {
                        bat "\"${env.DOCKER_EXE}\" build -t ${env.IMG} ."
                    }
                }
            }
        }

        // 3. Test Docker Image
        stage('Test') {
            steps {
                script {
                    if (isUnix()) {
                        sh "docker run --rm ${env.IMG} nginx -t"
                    } else {
                        bat "\"${env.DOCKER_EXE}\" run --rm ${env.IMG} nginx -t"
                    }
                }
            }
        }

        // 4. AWS Authentication
        stage('AWS Authentication') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'aws-secret',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret-key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    script {
                        if (isUnix()) {
                            sh '''
                                aws sts get-caller-identity
                            '''
                        } else {
                            bat '''
                                aws sts get-caller-identity
                            '''
                        }
                    }
                }
            }
        }

        // 5. Push Image to AWS ECR
        stage('Push to AWS ECR') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'aws-secret',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret-key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    script {

                        if (isUnix()) {
                            sh """
                                aws ecr get-login-password --region ${env.REGION} | docker login --username AWS --password-stdin ${env.REGISTRY}

                                docker tag ${env.IMG} ${env.VERSIONED}
                                docker tag ${env.IMG} ${env.REMOTE}

                                docker push ${env.VERSIONED}
                                docker push ${env.REMOTE}
                            """
                        } else {
                            bat """
                                aws ecr get-login-password --region ${env.REGION} | "${env.DOCKER_EXE}" login --username AWS --password-stdin ${env.REGISTRY}

                                "${env.DOCKER_EXE}" tag ${env.IMG} ${env.VERSIONED}
                                "${env.DOCKER_EXE}" tag ${env.IMG} ${env.REMOTE}

                                "${env.DOCKER_EXE}" push ${env.VERSIONED}
                                "${env.DOCKER_EXE}" push ${env.REMOTE}
                            """
                        }

                        echo 'Docker image pushed successfully to AWS ECR!'
                    }
                }
            }
        }

        // 6. Optional EC2 Deployment
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
                        credentialsId: 'aws-secret',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'aws-secret-key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    script {
                        if (isUnix()) {
                            sh """
                                aws ssm send-command \
                                --region ${env.REGION} \
                                --instance-ids ${env.INSTANCE} \
                                --document-name AWS-RunShellScript \
                                --parameters file://deploy.json
                            """
                        } else {
                            bat """
                                aws ssm send-command --region ${env.REGION} --instance-ids ${env.INSTANCE} --document-name AWS-RunPowerShellScript --parameters file://deploy.json
                            """
                        }
                    }
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
            echo 'Jenkins Pipeline Execution Completed.'
        }
    }
}
