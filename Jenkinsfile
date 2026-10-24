pipeline {
    agent any
 options {
        timestamps()
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'REGISTRY',
            choices: ['DOCKERHUB', 'ECR'],
            description: 'Choose Docker Hub or AWS ECR'
        )
    }

    environment {
        // Configure these values for your Jenkins setup
        SONAR_SERVER = 'SonarQube'
        SONAR_PROJECT_KEY = 'sonarqube-demo'

        // Docker Hub settings
        DOCKERHUB_CREDENTIALS = 'dockerhub-creds'
        DOCKERHUB_REPO = 'ninande/my-web-app'

        // AWS ECR settings
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '699258777180'
        ECR_REPO = 'my-web-app'

        // Set by the Docker build stage
        IMAGE_TAG = ''
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                 sh '''
                    echo "Source code checked out"
                    git log -1 --oneline
                '''
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
        stage('Unit Tests') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    junit(
                        testResults: 'target/surefire-reports/TEST-*.xml',
                        allowEmptyResults: true
                    )
                }
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
       
stage('Docker Build') {
    steps {
        script {
            env.IMAGE_NAME = 'ninande/my-web-app'
            env.IMAGE_TAG = "${env.BUILD_NUMBER}"

            sh '''
                set -eu

                echo "Building image: ${IMAGE_NAME}:${IMAGE_TAG}"

                docker build \
                  -t "${IMAGE_NAME}:${IMAGE_TAG}" \
                  -t "${IMAGE_NAME}:latest" .
            '''
        }
    }
}

        stage('Trivy Image Vulnerability Scan') {
            steps {
                sh '''
                    mkdir -p reports

                    set +e
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      --exit-code 1 \
                      --format json \
                      --output reports/trivy-image-report.json \
                      "${IMAGE_NAME}:${IMAGE_TAG}"
                    scan_status=$?
                    set -e

                    echo "Trivy scan exit code: ${scan_status}"
                    echo "JSON report: reports/trivy-image-report.json"

                    exit "$scan_status"
                '''
            }
        }

        stage('Push to Docker Registry') {
            steps {
                script {
                    if (params.REGISTRY == 'DOCKERHUB') {
                        withCredentials([
                            usernamePassword(
                                credentialsId: env.DOCKERHUB_CREDENTIALS,
                                usernameVariable: 'DH_USER',
                                passwordVariable: 'DH_TOKEN'
                            )
                        ]) {
                            sh '''
                                set +x
                                echo "$DH_TOKEN" | docker login \
                                  -u "$DH_USER" --password-stdin

                                docker push "${IMAGE_NAME}:${IMAGE_TAG}"
                                docker push "${IMAGE_NAME}:latest"

                                docker logout
                            '''
                        }
                    } else {
                        sh '''
                            set -eu

                            aws ecr get-login-password \
                              --region "${AWS_REGION}" |
                              docker login \
                                --username AWS \
                                --password-stdin \
                                "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

                            docker push "${IMAGE_NAME}:${IMAGE_TAG}"
                            docker push "${IMAGE_NAME}:latest"

                            docker logout \
                              "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        '''
                    }
                }
            }
        }
        
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'reports/*.json',
                allowEmptyArchive: true,
                fingerprint: true
            )

            echo "Pipeline result: ${currentBuild.currentResult}"
        }

        success {
            echo 'Build, tests, scans, and image publishing completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage and archived reports.'
        }

        cleanup {
            sh 'docker image prune -f || true'
        }
    }
}
