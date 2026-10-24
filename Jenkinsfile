
pipeline {
    agent any

    parameters {
        choice(
            name: 'VERSION',
            choices: ['v1', 'v2', 'v3'],
            description: 'Select To-Do App version'
        )
    }

    environment {
        IMAGE_NAME = 'praneeth2006racham/todo-app'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    docker build -t "$env:IMAGE_NAME`:$env:VERSION" .

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker build failed"
                    }
                '''
            }
        }

        stage('Login and Push to DockerHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    powershell '''
                        $env:DH_TOKEN |
                            docker login -u $env:DH_USER --password-stdin

                        if ($LASTEXITCODE -ne 0) {
                            throw "DockerHub login failed"
                        }

                        docker push "$env:IMAGE_NAME`:$env:VERSION"

                        if ($LASTEXITCODE -ne 0) {
                            docker logout
                            throw "Docker push failed"
                        }

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-file',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    powershell '''
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        kubectl apply -f "k8s/$env:VERSION.yaml"

                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes deployment failed"
                        }

                        kubectl rollout status "deployment/todo-$env:VERSION"

                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes rollout failed"
                        }
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'To-Do App pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
