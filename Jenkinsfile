pipeline {
  agent any

  environment {
    AWS_REGION = 'ap-southeast-2'
    ECR_REPO = '730335329548.dkr.ecr.ap-southeast-2.amazonaws.com/translator-api'
    CLUSTER_NAME = 'translator-cluster'
    SERVICE_NAME = 'translator-service'
    TASK_FAMILY = 'translator-task'
    CONTAINER_NAME = 'translator-api'
    CONTAINER_PORT = '8000'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'dev', url: 'https://github.com/apeCry/translator-api.git'
      }
    }

    stage('Docker Build & Push') {
      steps {
        script {
          IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

          withCredentials([usernamePassword(
            credentialsId: 'aws-credentials',
            usernameVariable: 'AWS_ACCESS_KEY_ID',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
          )]) {
            sh '''
              export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
              export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO

              docker build -t $ECR_REPO:$IMAGE_TAG .
              docker push $ECR_REPO:$IMAGE_TAG
            '''
          }
        }
      }
    }

    stage('Deploy to ECS') {
      steps {
        script {
          def envContent = readFile '.env'
          def envList = envContent
            .split('\n')
            .findAll { it && !it.startsWith('#') }
            .collect {
              def (key, value) = it.split('=', 2)
              [name: key, value: value]
            }

          def taskDef = [
            family: env.TASK_FAMILY,
            networkMode: 'awsvpc',
            requiresCompatibilities: ['FARGATE'],
            cpu: '512',
            memory: '1024',
            executionRoleArn: 'arn:aws:iam::730335329548:role/ecsTaskExecutionRoleTerraform',
            containerDefinitions: [[
              name: env.CONTAINER_NAME,
              image: "${env.ECR_REPO}:${IMAGE_TAG}",
              essential: true,
              portMappings: [[
                containerPort: env.CONTAINER_PORT.toInteger(),
                hostPort: env.CONTAINER_PORT.toInteger(),
                protocol: 'tcp'
              ]],
              environment: envList
            ]]
          ]

          writeJSON file: 'taskdef.json', json: taskDef, pretty: 4

          withCredentials([usernamePassword(
            credentialsId: 'aws-credentials',
            usernameVariable: 'AWS_ACCESS_KEY_ID',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
          )]) {
            sh '''
              export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
              export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

              aws ecs register-task-definition --cli-input-json file://taskdef.json

              LATEST_REVISION=$(aws ecs list-task-definitions --family-prefix $TASK_FAMILY --sort DESC --region $AWS_REGION --query "taskDefinitionArns[0]" --output text)

              aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME \
                --task-definition $LATEST_REVISION \
                --force-new-deployment --region $AWS_REGION
            '''
          }
        }
      }
    }
  }
}
