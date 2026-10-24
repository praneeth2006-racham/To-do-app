
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

                        git fetch origin v3
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to fetch v3 branch"
                        }

                        $yamlFile = "$env:TEMP\\$env:VERSION.yaml"

                        git show "FETCH_HEAD:k8s/$env:VERSION.yaml" |
                            Set-Content -Path $yamlFile

                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to retrieve Kubernetes YAML"
                        }

                        kubectl apply -f $yamlFile
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
