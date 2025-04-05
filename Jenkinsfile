pipeline {
  agent any

  environment {
    AWS_REGION = 'ap-southeast-2'
    ECR_REPO   = '730335329548.dkr.ecr.ap-southeast-2.amazonaws.com/translator-api'
    CLUSTER    = 'translator-cluster'
    SERVICE    = 'translator-service'
    TASK_FAMILY = 'translator-task'
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
          def IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-credentials'
          ]]) {
            sh '''
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
              docker build -t $ECR_REPO:$IMAGE_TAG .
              docker push $ECR_REPO:$IMAGE_TAG
            '''
          }

          env.IMAGE_TAG = IMAGE_TAG
        }
      }
    }

    stage('Deploy to ECS') {
      environment {
        PORT             = credentials('env-port')
        DATABASE_URL     = credentials('env-database-url')
        API_PREFIX       = credentials('env-api-prefix')
        SWAGGER_DOC_PATH = credentials('env-swagger-doc-path')
        JWT_SECRET       = credentials('env-jwt-secret')
      }

      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-credentials'
        ]]) {
          script {
            def taskDef = [
              family: env.TASK_FAMILY,
              networkMode: 'awsvpc',
              requiresCompatibilities: ['FARGATE'],
              cpu: '512',
              memory: '1024',
              executionRoleArn: 'arn:aws:iam::730335329548:role/ecsTaskExecutionRoleTerraform',
              containerDefinitions: [[
                name: 'translator-api',
                image: "${env.ECR_REPO}:${env.IMAGE_TAG}",
                essential: true,
                portMappings: [[
                  containerPort: 8000,
                  hostPort: 8000,
                  protocol: 'tcp'
                ]],
                environment: [
                  [name: 'PORT',             value: env.PORT],
                  [name: 'DATABASE_URL',     value: env.DATABASE_URL],
                  [name: 'API_PREFIX',       value: env.API_PREFIX],
                  [name: 'SWAGGER_DOC_PATH', value: env.SWAGGER_DOC_PATH],
                  [name: 'JWT_SECRET',       value: env.JWT_SECRET]
                ]
              ]]
            ]

            writeJSON file: 'taskdef.json', json: taskDef, pretty: 4

            sh '''
              aws ecs register-task-definition --cli-input-json file://taskdef.json
              LATEST_REVISION=$(aws ecs describe-task-definition --task-definition $TASK_FAMILY --query "taskDefinition.revision" --output text)
              echo "Using latest task definition: $TASK_FAMILY:$LATEST_REVISION"
              aws ecs update-service \
                --cluster $CLUSTER \
                --service $SERVICE \
                --task-definition $TASK_FAMILY:$LATEST_REVISION \
                --force-new-deployment
            '''
          }
        }
      }
    }
  }
}
