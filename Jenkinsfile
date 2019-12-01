pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME = "juanpab/vulnerable-app"
    }
    stages {
        stage('SonarQube Analysis') {
            steps {
                script {
                  // debe estar configurado en Global Tool Configuration
                  scannerHome = tool 'sonarscanner'
                }
                withSonarQubeEnv('sonarqube') {
                  sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=application-test"
                }
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('ScanDockerImage') {
            when {
                branch 'master'
            }
            steps {
                aquaMicroscanner imageName: DOCKER_IMAGE_NAME, notCompliesCmd: '', onDisallowed: 'ignore', outputFormat: 'html'
            }
        }
        stage('PushDockerImage') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'vulnerable-app-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 0
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'vulnerable-app-kube-canary.yml',
                    enableConfigSubstitution: true
                )
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'vulnerable-app-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}