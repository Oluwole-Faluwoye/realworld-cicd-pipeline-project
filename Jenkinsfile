def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]

pipeline {
    agent any
    environment { 
        WORKSPACE = "${env.WORKSPACE}" 
        NEXUS_PRIVATE_IP = '172.31.38.223' // Replace with your Nexus EC2 private IP
    }
    tools { 
        maven 'localMaven'; 
        jdk 'localJdk' 
    }

    // Helper function to run Maven safely
    def runMaven = { mavenGoal ->
        withCredentials([usernamePassword(credentialsId: 'Nexus-Credential', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            configFileProvider([configFile(fileId: 'maven-settings-template', variable: 'MAVEN_SETTINGS')]) {
                script {
                    TMP_SETTINGS = "${env.WORKSPACE}/tmp-settings.xml"
                    try {
                        sh """
                        cp \$MAVEN_SETTINGS $TMP_SETTINGS
                        sed -i 's|\\\${username}|$NEXUS_USER|g' $TMP_SETTINGS
                        sed -i 's|\\\${password}|$NEXUS_PASS|g' $TMP_SETTINGS
                        sed -i 's|\\\${nexus_private_ip}|$NEXUS_PRIVATE_IP|g' $TMP_SETTINGS
                        mvn $mavenGoal --settings $TMP_SETTINGS
                        """
                    } finally {
                        sh "rm -f $TMP_SETTINGS"
                    }
                }
            }
        }
    }

    stages {
        stage('Build') { 
            steps { runMaven('clean package') } 
            post { success { archiveArtifacts artifacts: '**/*.war' } } 
        }

        stage('Unit Test') { steps { runMaven('test') } }

        stage('Integration Test') { steps { runMaven('verify -DskipUnitTests') } }

        stage('Checkstyle Analysis') { 
            steps { runMaven('checkstyle:checkstyle') } 
            post { success { echo 'Generated Checkstyle Analysis Result' } } 
        }

        stage('SonarQube Inspection') {
            steps {
                withCredentials([string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')]) {
                    runMaven("sonar:sonar -Dsonar.projectKey=Java-WebApp-Project -Dsonar.host.url=http://172.31.1.44:9000 -Dsonar.login=$SONAR_TOKEN")
                }
            }
        }

        stage('SonarQube GateKeeper') { 
            steps { timeout(time: 1, unit: 'HOURS') { waitForQualityGate abortPipeline: true } } 
        }

        stage('Nexus Artifact Uploader') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Nexus-Credential', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_PRIVATE_IP}:8081",
                        groupId: 'webapp',
                        version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                        repository: 'maven-project-releases',
                        credentialsId: 'Nexus-Credential',
                        artifacts: [[
                            artifactId: 'webapp', 
                            classifier: '', 
                            file: "${WORKSPACE}/webapp/target/webapp.war", 
                            type: 'war'
                        ]]
                    )
                }
            }
        }

        stage('Deploy to Development') { 
            environment { HOSTS='dev' } 
            steps { deployAnsible(HOSTS) } 
        }

        stage('Deploy to Staging') { 
            environment { HOSTS='stage' } 
            steps { deployAnsible(HOSTS) } 
        }

        stage('QA Approval') { 
            steps { input('Do you want to proceed to Production?') } 
        }

        stage('Deploy to Production') { 
            environment { HOSTS='prod' } 
            steps { deployAnsible(HOSTS) } 
        }
    }

    post {
        always {
            slackSend channel: '#af-cicd-pipeline',
                      color: COLOR_MAP[currentBuild.currentResult],
                      message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' Build #${env.BUILD_NUMBER}\nTimestamp: ${env.BUILD_TIMESTAMP}\nWorkspace: ${env.WORKSPACE}\nMore info: ${env.BUILD_URL}"
        }
    }
}

// ---------------- Helper Function ----------------
def deployAnsible(hosts) {
    withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
        sh """
        ansible-playbook \
        -i ${WORKSPACE}/ansible-config/aws_ec2.yaml \
        ${WORKSPACE}/deploy.yaml \
        --extra-vars "ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$hosts workspace_path=$WORKSPACE"
        """
    }
}
