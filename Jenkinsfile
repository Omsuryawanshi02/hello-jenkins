
def run(cmd) {
    if (isUnix()) {
        sh cmd
    } else {
        bat cmd
    }
}

def dockerRun(cmd) {
    if (isUnix()) {
        sh "docker ${cmd}"
    } else {
        bat "\"C:\\Users\\om\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe\" ${cmd}"
    }
}

def awsRun(cmd) {
    withCredentials([
        string(
            credentialsId: 'aws-key-id',
            variable: AKIAXU3PVMT5ZA33QRH4
        ),
        string(
            credentialsId: 'aws-secret',
            variable: wZcmyhLcdHx+UPz816l1atMgKGYZ+Ts6XPUluccy
        )
    ]) {
        run(cmd)
    }
}

def ecrLogin() {
    awsRun(
        "aws ecr get-login-password --region ${env.REGION} | " +
        "docker login --username AWS --password-stdin ${env.REGISTRY}"
    )
}

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
        REGION   = 'ap-south-1'
        ACCOUNT  = '525830808827'
        REPO     = 'hello-jenkins'
        INSTANCE = 'none'

        REGISTRY = "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com"
        IMG      = "${REPO}:${BUILD_NUMBER}"
        REMOTE   = "${REGISTRY}/${REPO}:latest"
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
                    dockerRun("build -t ${IMG} .")
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    dockerRun("run --rm ${IMG} nginx -t")
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

                    dockerRun("tag ${IMG} ${REMOTE}")
                    dockerRun("push ${REMOTE}")
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
}
