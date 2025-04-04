pipeline {
  agent any

  environment {
    AWS_REGION = "ap-southeast-2"
    ECR_REPO = "730335329548.dkr.ecr.ap-southeast-2.amazonaws.com/translator-api"
    ECS_CLUSTER = "translator-cluster"
    ECS_SERVICE = "translator-service"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.IMAGE_TAG = imageTag
          sh "docker build -t translator-api:${IMAGE_TAG} ."
        }
      }
    }

    stage('Push to ECR') {
      steps {
        script {
          sh """
            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
            docker tag translator-api:${IMAGE_TAG} $ECR_REPO:${IMAGE_TAG}
            docker push $ECR_REPO:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Deploy to ECS') {
      steps {
        script {
          sh """
            aws ecs update-service \
              --cluster $ECS_CLUSTER \
              --service $ECS_SERVICE \
              --force-new-deployment \
              --region $AWS_REGION
          """
        }
      }
    }
  }
}
