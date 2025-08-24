// ---------- SonarQube with Java 11 ----------
stage('SonarQube Inspection') {
    steps {
        // Manually set JAVA_HOME to Java 11 installation
        withEnv([
            "JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto.x86_64",
            "PATH=/usr/lib/jvm/java-11-amazon-corretto.x86_64/bin:${env.PATH}"
        ]) {
            ansiColor('xterm') {
                script {
                    sh 'java -version'
                    sh 'echo "JAVA_HOME is $JAVA_HOME"'
                    withCredentials([string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            if [ -z "$SONAR_TOKEN" ]; then
                                echo "ERROR: Sonar token is empty!"
                                exit 1
                            fi
                            echo "Sonar token is present (masked)."
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
