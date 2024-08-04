pipeline {
    environment {
        NodejsHome = tool "myNode"
        dockerHome = tool "myDocker"
        SonarQubeHome = tool "mySonar"
        OWASP_HOME = tool "myDp"
        NVD_KEY = credentials('NVD_KEY')
        TMDB_V3_API_KEY = credentials('TMDB_V3_API_KEY')
        SONARQUBE_TOKEN = credentials('SonarNetflix')
        PATH = "${dockerHome}/bin:${NodejsHome}/bin:${SonarQubeHome}/bin:${OWASP_HOME}/bin:${PATH}"
    }
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Install Dependencies') {
            steps {
                script {
                    echo 'Installing npm dependencies...'
                    sh 'npm install'
                }
            }
        }
        stage('OWASP Dependency-Check') {
            steps {
                script {
                    echo 'Running OWASP Dependency-Check...'
                    sh '''
                    #!/bin/bash
                    set -x  # Enable debug mode
                    dependency-check.sh --project "Netlifex" --scan . --nvdApiKey=${NVD_KEY} --format XML --out ./reports || {
                        echo "OWASP Dependency-Check failed"
                        exit 1
                    }
                    '''
                }
            }
            post {
                always {
                    script {
                        echo 'Listing report files for debugging...'
                        sh 'ls -la ./reports'
                        
                        echo 'Inspecting XML file for issues...'
                        sh 'head -n 20 ./reports/dependency-check-report.xml'
                        sh 'tail -n 20 ./reports/dependency-check-report.xml'
                    }
                    dependencyCheckPublisher pattern: './reports/dependency-check-report.xml'
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    echo 'Running SonarQube analysis...'
                    withSonarQubeEnv('mySonar') {
                        sh 'sonar-scanner -Dsonar.projectKey=Netflix -Dsonar.sources=. -Dsonar.host.url=http://sonarqube:9000 -Dsonar.login=${SONARQUBE_TOKEN}'
                    }
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    def imageTag = "${env.BUILD_NUMBER}"
                    dockerImage = docker.build(
                        "seifseddik120/netflix-2024:${imageTag}",
                        "--build-arg TMDB_V3_API_KEY=${env.TMDB_V3_API_KEY} ."
                    )
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    echo 'Pushing Docker image to DockerHub...'
                    docker.withRegistry('', 'DockerHub') {
                        dockerImage.push()
                    }
                }
            }
        }
        stage('Start Minikube') {
            steps {
                script {
                    echo 'Starting Minikube...'
                    sh 'minikube start --driver=docker'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo 'Deploying to Kubernetes...'
                    sh 'envsubst < deployment.yaml | kubectl apply -f -'
                    sh 'kubectl apply -f service.yaml'
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                script {
                    echo 'Verifying deployment...'
                    sh 'kubectl get pods'
                    sh 'kubectl get services'
                }
            }
        }
    }
    post {
        always {
            emailext(
                attachLog: true,
                subject: "Jenkins Pipeline: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
                body: "Build details: ${env.BUILD_URL}",
                to: 'seifhesham2030@gmail.com'
            )
        }
    }
}
