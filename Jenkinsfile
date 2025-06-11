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
                            -Dsonar.token=$SONAR_TOKEN
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
        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('TRIVY IMAGE SCAN') {
            steps {
                sh 'trivy image -o trivy-image-result.html $IMAGE_NAME'
            }
        }
    }
    post{
        always{
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: './', reportFiles: 'trivy-fs-result.html', reportName: 'trivy fs HTML Report', reportTitles: '', useWrapperFileDirectly: true])
        }
    }
}
