pipeline {
    agent any

    environment {
        APP_NAME = "webapp-java"
        APP_IMAGE = "webapp-java:${BUILD_NUMBER}"
        IMAGE_ARCHIVE = "build/webapp-java-${BUILD_NUMBER}.tar"
        ANSIBLE_INVENTORY = "/var/lib/jenkins/ansible/inventaire"
        SONAR_HOST_URL = "http://127.0.0.1:9000"
    }

    stages {
        stage("Planifier") {
            steps {
                echo "Jira porte la planification: epics, user stories, bugs et sprint backlog."
            }
        }

        stage("Coder") {
            steps {
                echo "GitHub declenche ce pipeline via commit/push ou webhook Jenkins."
                sh "git rev-parse --short HEAD"
            }
        }

        stage("Compiler") {
            steps {
                sh "mvn clean compile"
            }
        }

        stage("Tester") {
            steps {
                sh "mvn test"
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: "**/target/surefire-reports/*.xml"
                }
            }
        }

        stage("Qualite") {
            steps {
                withCredentials([string(credentialsId: "sonarqube-token", variable: "SONAR_TOKEN")]) {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.host.url="$SONAR_HOST_URL" \
                          -Dsonar.token="$SONAR_TOKEN"
                    '''
                }
            }
        }

        stage("Packaging Maven") {
            steps {
                sh "mvn package"
                archiveArtifacts artifacts: "webapp/target/webapp.war", fingerprint: true
            }
        }

        stage("Packaging Docker") {
            steps {
                sh '''
                    mkdir -p build
                    docker build -t "$APP_IMAGE" .
                    docker save "$APP_IMAGE" -o "$IMAGE_ARCHIVE"
                '''
                archiveArtifacts artifacts: "build/*.tar", fingerprint: true
            }
        }

        stage("Securite") {
            steps {
                sh '''
                    if command -v trivy >/dev/null 2>&1; then
                      trivy image --severity HIGH,CRITICAL --exit-code 0 "$APP_IMAGE"
                    else
                      echo "Trivy non installe: installer Trivy sur l agent Jenkins pour activer le scan."
                    fi
                '''
            }
        }

        stage("Deployer") {
            steps {
                sh '''
                    ansible-playbook -i "$ANSIBLE_INVENTORY" ansible/deploy-k8s.yaml \
                      -e app_image="$APP_IMAGE" \
                      -e image_archive="$WORKSPACE/$IMAGE_ARCHIVE"
                '''
            }
        }
    }

    post {
        always {
            sh "docker image ls $APP_NAME || true"
        }
    }
}
