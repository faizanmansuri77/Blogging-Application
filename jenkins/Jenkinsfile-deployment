pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool('sonar-scanner')
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/faizanmansuri77/Blogging-Application.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('File System Scan') {
            steps {
                sh 'trivy fs . > trivy-fs-report.html'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=bloggingapp \
                    -Dsonar.projectKey=bloggingapp \
                    -Dsonar.java.binaries=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate(
                        abortPipeline: false,
                        credentialsId: 'sonar-token'
                    )
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-settings',
                    jdk: 'jdk17',
                    maven: 'maven3',
                    traceability: true
                ) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: 'docker',
                        toolName: 'docker'
                    ) {
                        sh 'docker build -t orionpax77/bloggingapp:latest .'
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh 'trivy image orionpax77/bloggingapp:latest > trivy-image-report.html'
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: 'docker',
                        toolName: 'docker'
                    ) {
                        sh 'docker push orionpax77/bloggingapp:latest'
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    clusterName: 'orionpax77-cluster',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://YOUR-EKS-ENDPOINT.ap-south-1.eks.amazonaws.com'
                ) {
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    clusterName: 'orionpax77-cluster',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://YOUR-EKS-ENDPOINT.ap-south-1.eks.amazonaws.com'
                ) {
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
                }
            }
        }

    }

    post {
        always {
            script {

                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'

                def bannerColor = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

                def body = """
                <html>
                <body>
                    <div style="border:4px solid ${bannerColor};padding:10px">
                        <h2>${jobName} - Build ${buildNumber}</h2>

                        <div style="background-color:${bannerColor};padding:10px">
                            <h3 style="color:white">
                            Pipeline Status: ${pipelineStatus}
                            </h3>
                        </div>

                        <p>
                        Check Console:
                        <a href="${env.BUILD_URL}">
                        Build Logs
                        </a>
                        </p>

                    </div>
                </body>
                </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                    body: body,
                    to: 'abrahim.ctech@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-*.html'
                )
            }
        }
    }
}
