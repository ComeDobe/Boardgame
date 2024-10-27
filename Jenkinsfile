pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        JAVA_HOME = sh(script: 'readlink -f /usr/bin/java | sed "s:/bin/java::"', returnStdout: true).trim()
        MAVEN_HOME = tool 'maven'
        SCANNER_HOME = tool 'sonar-scanner'
        PATH = "${env.JAVA_HOME}/bin:${env.MAVEN_HOME}/bin:${env.SCANNER_HOME}/bin:${env.PATH}"
    }

    stages {
        stage('Verify Environment') {
            steps {
                sh 'echo JAVA_HOME: $JAVA_HOME'
                sh '$JAVA_HOME/bin/java -version'
                sh 'echo MAVEN_HOME: $MAVEN_HOME'
                sh '$MAVEN_HOME/bin/mvn -version'
                sh 'echo PATH: $PATH'
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/ComeDobe/Boardgame.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn -Dmaven.compiler.fork=true -Dmaven.compiler.executable=${JAVA_HOME}/bin/javac compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn test"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
    stage('SonarQube Analysis') {
        steps {
            withSonarQubeEnv('sonar') {
                sh """
                    echo "SonarQube URL: \${SONAR_HOST_URL}"
                    echo "Scanner Home: \${SCANNER_HOME}"
                    ${SCANNER_HOME}/bin/sonar-scanner -X -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame -Dsonar.java.binaries=.
                """
            }
        }
    }

        
stage('Quality Gate') {
    steps {
        timeout(time: 15, unit: 'MINUTES') {
            script {
                def qg = null
                retry(3) {
                    sleep(time: 10, unit: 'SECONDS')
                    echo "Tentative de vérification de la Quality Gate..."
                    qg = waitForQualityGate abortPipeline: false
                }
                if (qg == null) {
                    error "Impossible d'obtenir le statut de la Quality Gate après plusieurs tentatives"
                } else if (qg.status != 'OK') {
                    error "Quality Gate échec: ${qg.status}"
                } else {
                    echo "Quality Gate passée avec succès"
                }
            }
        }
    }
}
            
        stage('Build') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn package"
            }
        }


stage('Publish To Nexus') {
    steps {
        sh """
            echo "JAVA_HOME is set to: ${JAVA_HOME}"
            echo "PATH is set to: ${PATH}"
            ${JAVA_HOME}/bin/java -version
            ${MAVEN_HOME}/bin/mvn -version
            ${MAVEN_HOME}/bin/mvn clean deploy
            ${MAVEN_HOME}/bin/mvn deploy -X

        """
    }
}



        
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t cdobe01/boardshack:latest ."
                    }
                }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html cdobe01/boardshack:latest"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push cdobe01/boardshack:latest"
                    }
                }
            }
        }
        
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://172.31.8.146:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
        
        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://172.31.8.146:6443') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
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
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'christiandobe01@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
