
def dockerRun(String cmd) {
    if (isUnix()) {
        sh "docker ${cmd}"
    } else {
        bat "\"${env.DOCKER_EXE}\" ${cmd}"
    }
}

// Execute AWS commands using Jenkins-managed credentials
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
            bat '''
                aws configure set aws_access_key_id %AWS_ACCESS_KEY_ID%
                aws configure set aws_secret_access_key %AWS_SECRET_ACCESS_KEY%
            '''
        }
    }
}
    ]) {
        if (isUnix()) {
            sh cmd
        } else {
            bat cmd
        }
    }
}

// Authenticate Docker with AWS ECR
def ecrLogin() {
    def dockerCommand = isUnix()
        ? 'docker'
        : "\"${env.DOCKER_EXE}\""

    awsRun(
        "aws ecr get-login-password --region ${env.REGION} | " +
        "${dockerCommand} login --username AWS --password-stdin ${env.REGISTRY}"
    )
}

// Deploy application using AWS Systems Manager
def deploy() {
    awsRun(
        "aws ssm send-command --region ${env.REGION} " +
        "--instance-ids ${env.INSTANCE} " +
        "--document-name AWS-RunShellScript " +
        "--parameters file://deploy.json"
    )
}

pipeline {

    agent any

    environment {
        REGION     = 'eu-north-1'
        ACCOUNT    = '525830808827'
        REPO       = 'hello-jenkins'
        INSTANCE   = 'none'

        DOCKER_EXE = 'C:\\Users\\om\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'

        REGISTRY   = "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com"
        IMG        = "${REPO}:${BUILD_NUMBER}"
        REMOTE     = "${REGISTRY}/${REPO}:latest"
    }

    stages {

        stage('Docker Check') {
            steps {
                script {
                    dockerRun('--version')
                    dockerRun('info')
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    dockerRun("build -t ${env.IMG} .")
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    dockerRun("run --rm ${env.IMG} nginx -t")
                }
            }
        }

        stage('AWS Authentication') {
            steps {
                script {
                    awsRun('aws sts get-caller-identity')
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                script {
                    ecrLogin()

                    dockerRun("tag ${env.IMG} ${env.REMOTE}")
                    dockerRun("push ${env.REMOTE}")
                }
            }
        }

        stage('Deploy') {
            when {
                not {
                    environment name: 'INSTANCE', value: 'none'
                }
            }

            steps {
                script {
                    deploy()
                }
            }
        }
    }

    post {
        success {
            echo 'Jenkins pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the console output.'
        }
    }
}
