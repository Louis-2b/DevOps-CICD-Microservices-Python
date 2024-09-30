pipeline {
    agent any
    tools {
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        REPOSITORY = '192.168.222.60:8083'
        IMAGE_REPO = 'microservicepython'
    }
    stages {
        stage ('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage ('Checkout From Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Louis-2b/DevOps-CICD-Microservices-Python.git'
            }
        }
        stage ('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' 
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Microservices-Python \
                        -Dsonar.java.binaries=. \
                        -Dsonar.projectKey=Microservices-Python 
                    '''
                }
            }
        }
        stage ('quality gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SONAR-TOKEN'
                }
            }
        }
        stage ('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ('TRIVY File Scan') {
            steps {
                sh "trivy fs . > trivy-fs_report.txt"
            }
        }
        stage ('Docker Build') {
            steps {
                script {
                    sh "docker build -t ${REPOSITORY}/${IMAGE_REPO}:${BUILD_NUMBER} ."
                }
            }
        }
        stage ('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'NEXUS-TOKEN', passwordVariable: 'PSW', usernameVariable: 'USER')]) {
                        sh "echo ${PSW} | docker login -u ${USER} --password-stdin ${REPOSITORY}"
                        sh "docker push ${REPOSITORY}/${IMAGE_REPO}:${BUILD_NUMBER}"
                    }
                }
            }
        }
        stage('TRIVY') {
            steps {
                sh "trivy image ${REPOSITORY}/${IMAGE_REPO}:${BUILD_NUMBER} > trivyimage.txt"
            }
        }
        stage ('Deploy to Container') {
            steps {
                sh 'docker run -d --name microsvcpython -p 5000:5000 ${REPOSITORY}/${IMAGE_REPO}:${BUILD_NUMBER}'
            }
        }
        stage ('Deploy to kubernetes') {
            steps {
                dir ('kubernetes/development') {
                    script {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'K3S', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'kubectl apply -f deployment.yaml'
                            sh 'kubectl apply -f ingress.yaml'
                            sh 'kubectl apply -f service.yaml'
                        }
                    }
                }
                dir ('kubernetes/staging') {
                    script {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'K3S', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'kubectl apply -f deployment.yaml'
                            sh 'kubectl apply -f ingress.yaml'
                            sh 'kubectl apply -f service.yaml'
                        }
                    }
                }
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'stephanetchanga2b@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}