pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node18'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/imranafset/starbucks-app.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {     // <-- FIXED HERE
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=starbucks \
                        -Dsonar.projectKey=starbucks \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('TRIVY FS Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t starbucks ."
                        sh "docker tag starbucks faizalaccount/starbucks:latest"
                        sh "docker push faizalaccount/starbucks:latest"
                    }
                }
            }
        }

        stage('TRIVY Image Scan') {
            steps {
                sh "trivy image faizalaccount/starbucks:latest > trivyimage.txt"
            }
        }

        stage('Deploy App to Docker Container') {
            steps {
                sh 'docker rm -f starbucks || true'
                sh 'docker run -d --name starbucks -p 3000:3000 faizalaccount/starbucks:latest'
            }
        }
    }

    post {
        always {
            dir('.') {
                script {
                    def buildStatus = currentBuild.currentResult
                    def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'GitHub User'

                    emailext(
                        subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: """
                            <p>This is an automated Jenkins CI/CD pipeline report.</p>
                            <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                            <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                            <p><strong>Build Status:</strong> ${buildStatus}</p>
                            <p><strong>Started by:</strong> ${buildUser}</p>
                            <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                        """,
                        to: 'mi3081557@gmail.com',
                        mimeType: 'text/html',
                        attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                    )
                }
            }
        }
    }
}
