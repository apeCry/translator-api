pipeline {
  agent any

  environment {
    AWS_REGION = "ap-southeast-2"
    ECR_REPO = "730335329548.dkr.ecr.ap-southeast-2.amazonaws.com/translator-api"
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
          IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.IMAGE_TAG = IMAGE_TAG
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

    stage('Terraform Apply') {
      steps {
        dir('terraform') {
          withCredentials([
            file(credentialsId: 'aws-creds', variable: 'AWS_CREDS'),
            file(credentialsId: 'ifa-env-file', variable: 'ENV_FILE')
          ]) {
            sh """
              export AWS_SHARED_CREDENTIALS_FILE=$AWS_CREDS
              terraform init
              terraform apply -auto-approve -var='image_tag=${IMAGE_TAG}'
            """
          }
        }
      }
    }
  }
}
