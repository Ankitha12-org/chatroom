pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        IMAGE_NAME = 'ankii1212/chatroom:latest'
        SCANNER_HOME = tool 'sonarqube-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile -DskipTests'
            }
        }
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }
        stage('trivy file scan'){
            steps{
                sh 'trivy fs --severity HIGH,CRITICAL --format json -o trivy-fs-result.json .'
            }
            post{
                always{
                    sh ''' trivy convert \
                    --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                    -o trivy-fs-result.html trivy-fs-result.json '''
                }
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey=' + NVD_API_KEY, odcInstallation: 'DP-Check'
                }
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-cred', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonar-server') {
                        sh '''#!/bin/bash
                            $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectKey=chatroom \
                            -Dsonar.projectName=chatroom \
                            -Dsonar.sources=. \
                        '''
                    }
                }
            }
        }
        stage('Sonarqube Quality Gate'){
            steps{
                timeout(time: 60, unit: 'SECONDS') {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-cred'
                }
            }
        }
        stage('Nexus Artifactory'){
            steps{
                withMaven(globalMavenSettingsConfig: 'maven', jdk: '', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy -DskipTests'
                }
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('TRIVY IMAGE SCAN') {
            steps {
                sh 'trivy image --format json -o trivy-image-result.json $IMAGE_NAME'
            }
            post{
                always{
                    sh ''' trivy convert \
                    --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                    -o trivy-image-result.html trivy-image-result.json '''
                }
            }
        }
         stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh 'echo $IMAGE_NAME'
                    sh 'docker push $IMAGE_NAME'
                }
            }
        }
        stage('Deploy to ec2'){
            steps {
                sshagent(['ssh-key']) {
                    withAWS(credentials: 'aws-cred', region: 'us-east-1') {
                        sh ''' 
                            ssh -o StrictHostKeyChecking=no ubuntu@3.89.36.80 "
                                docker stop chatroom-app || true
                                docker rm chatroom-app || true
                                docker rmi $(docker images -q) || true
                            
                                docker run --rm -itd --name chatroom-app -p 8080:8080 $IMAGE_NAME
                            "
                        '''    
                    }
                }
            }
        }
    }
    post{
        always{
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: './', reportFiles: 'trivy-fs-result.html', reportName: 'trivy fs HTML Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: './', reportFiles: 'trivy-image-result.html', reportName: 'trivy image HTML Report', reportTitles: '', useWrapperFileDirectly: true])
        }
    }  
}
