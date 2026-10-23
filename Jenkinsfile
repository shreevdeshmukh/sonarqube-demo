pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Trivy Security Scan') {
            steps {
                sh '''
                    trivy fs \
                      --scanners vuln,secret,misconfig \
                      --format json \
                      --output trivy-report.json \
                      .
                '''
            }
        }
        
        stage('Build and Test') {
            steps {
                sh 'mvn -B clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar \
                          -Dsonar.projectKey=sonarqube-demo \
                          -Dsonar.projectName=SonarQube-Demo \
                          -Dsonar.host.url="$SONAR_HOST_URL"
                    '''
                }
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true,
                  testResults: '**/target/surefire-reports/*.xml'

            archiveArtifacts allowEmptyArchive: true,
                             artifacts: 'trivy-report.json,target/*.jar'
        }
    }
}
