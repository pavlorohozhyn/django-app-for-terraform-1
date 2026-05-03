pipeline {
    agent {
        kubernetes {
            serviceAccount 'jenkins-admin'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ['sleep']
    args: ['99d']
  - name: git
    image: alpine/git
    command: ['sleep']
    args: ['99d']
"""
        }
    }

   environment {
        ECR_REPO = "211125349493.dkr.ecr.us-east-1.amazonaws.com/dev-lesson-8-9-test"
        GITOPS_REPO = "github.com/PavloRohozhyn/django-app-for-terraform.git"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Build & Push to ECR') {
            steps {
                container('kaniko') {
                    // Kaniko dont need docker login, his use IRSA (IAM role)
                    sh """
                    /kaniko/executor --context `pwd` \
                        --dockerfile Dockerfile \
                        --destination ${ECR_REPO}:${IMAGE_TAG} \
                        --destination ${ECR_REPO}:latest
                    """
                }
            }
        }

        stage('Update GitOps Manifests') {
            steps {
                container('git') {
                    script {
                        withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                            sh """
                                git config --global user.email "jenkins@example.com"
                                git config --global user.name "Jenkins CI"
                                git clone -b lesson-8-9 https://\$GH_TOKEN@github.com/PavloRohozhyn/terraform.git temp_infra
                                cd temp_infra
                                FILE_PATH="lesson-8-9/charts/django-app/values.yaml"
                                if [ -f "\$FILE_PATH" ]; then
                                    echo "Update tag to : ${IMAGE_TAG}"
                                    sed -i "s/tag: .*/tag: \\"${IMAGE_TAG}\\"/" "\$FILE_PATH"
                                    git add "\$FILE_PATH"
                                    git commit -m "Update Django image to ${IMAGE_TAG} (Build #${BUILD_NUMBER}) [skip ci]"
                                    git push origin lesson-8-9
                                else
                                    echo "error: file \$FILE_PATH not found"
                                    find . -maxdepth 3 -not -path '*/.*'
                                    exit 1
                                fi
                            """
                        }
                    }
                }
            }
        }
    }
}

