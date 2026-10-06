pipeline {
    agent any
    tools { nodejs 'nodejs' }
    environment {
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO     = "${ECR_REGISTRY}/myrepo"
        IMAGE_TAG    = "${new Date().format('yyyy-MM-dd-HH.mm', TimeZone.getTimeZone('Asia/Kolkata'))}-${BUILD_NUMBER}"
    }
    stages {
        stage('clean workspace') { steps { cleanWs() } }
        stage('checkout') { steps { checkout scm } }
        stage('build npm app') { steps { sh 'npm install' } }
        stage('unit test') { steps { sh 'npm test' } }
        stage('CQA') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        ${tool 'sonar'}/bin/sonar-scanner \
                        -Dsonar.projectKey=eks-app \
                        -Dsonar.projectName=eks-app \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=node_modules/**,public/**,images/**
                    """
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') { waitForQualityGate abortPipeline: true }
            }
        }
        stage('build image') { steps { sh 'docker build -t $ECR_REPO:$IMAGE_TAG .' } }
        stage('push image') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                    docker push $ECR_REPO:$IMAGE_TAG
                '''
            }
        }
        stage('update manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        sed -i "s|image: .*|image: $ECR_REPO:$IMAGE_TAG|" Manifests/deploy.yml
                        git config user.email "jenkins@example.com"
                        git config user.name "jenkins"
                        git add Manifests/deploy.yml
                        git commit -m "Update image to $IMAGE_TAG [skip ci]"
                        git push https://$GIT_USER:$GIT_TOKEN@github.com/SrikanthKotipalli0209/EKS-3tire_Application.git HEAD:main
                    '''
                }
            }
        }
    }
}
