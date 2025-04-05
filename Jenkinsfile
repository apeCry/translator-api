pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-southeast-2'
        ECR_REPOSITORY = '730335329548.dkr.ecr.ap-southeast-2.amazonaws.com/translator-api'
        ECS_CLUSTER = 'translator-cluster'
        ECS_SERVICE = 'translator-service'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

                    sh """
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPOSITORY
                        docker build -t $ECR_REPOSITORY:$IMAGE_TAG .
                        docker push $ECR_REPOSITORY:$IMAGE_TAG
                    """
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'ifa-env-file', variable: 'ENV_FILE')]) {
                        def envVars = readFile(ENV_FILE).split("\n").collect { 
                            def parts = it.split("=")
                            [name: parts[0], value: parts[1]]
                        }

                        def taskDef = [
                            family: "translator-task",
                            networkMode: "awsvpc",
                            requiresCompatibilities: ["FARGATE"],
                            cpu: "512",
                            memory: "1024",
                            executionRoleArn: "arn:aws:iam::730335329548:role/ecsTaskExecutionRoleTerraform",  // ✅ 使用 terraform 创建的角色名
                            containerDefinitions: [
                                [
                                    name: "translator-api",
                                    image: "$ECR_REPOSITORY:$IMAGE_TAG",
                                    portMappings: [[containerPort: 8000]],
                                    environment: envVars,
                                    essential: true
                                ]
                            ]
                        ]

                        writeJSON file: 'task-def.json', json: taskDef

                        sh '''
                            aws ecs register-task-definition --cli-input-json file://task-def.json
                            aws ecs update-service --cluster $ECS_CLUSTER --service $ECS_SERVICE --force-new-deployment
                        '''
                    }
                }
            }
        }
    }
}

