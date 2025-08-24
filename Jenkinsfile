pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://172.31.6.142:9000'
    }

    tools {
        jdk 'jdk11'  // Explicitly use Java 11 for Sonar
    }

    stages {
        stage('SonarQube Inspection') {
            steps {
                withEnv(["JAVA_HOME=${tool 'jdk11'}", "PATH=${tool 'jdk11'}/bin:${env.PATH}"]) {
                    ansiColor('xterm') {
                        script {
                            // Confirm Java version
                            sh 'java -version'
                            sh 'echo "JAVA_HOME is $JAVA_HOME"'

                            // Use Sonar token
                            withCredentials([string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')]) {
                                sh '''
                                    if [ -z "$SONAR_TOKEN" ]; then
                                        echo "ERROR: Sonar token is empty!"
                                        exit 1
                                    fi
                                    echo "Sonar token is present (masked in logs)."
                                '''
                                sh """
                                    mvn sonar:sonar \
                                        -Dsonar.projectKey=Java-WebApp-Project \
                                        -Dsonar.host.url=${SONAR_HOST_URL} \
                                        -Dsonar.login=\$SONAR_TOKEN \
                                        -X
                                """
                            }
                        }
                    }
                }
            }
        }
    }
}
