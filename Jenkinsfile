pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
mvn -B 
org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988 
-Dsonar.projectKey=sonarqube-demo 
-Dsonar.projectName=SonarQube-Demo 
-Dsonar.host.url="$SONAR_HOST_URL" 
-Dsonar.token="$SONAR_AUTH_TOKEN"
'''
}
}
}
    }

    post {
        always {
            archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
            junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
        }
    }
}
