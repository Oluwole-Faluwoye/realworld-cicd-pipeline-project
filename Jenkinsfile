pipeline {
    agent any

    environment {
        WORKSPACE      = "${env.WORKSPACE}"
        GIT_REPO       = credentials('GIT_REPO')
        NEXUS_URL      = credentials('NEXUS_URL')
        SONAR_HOST_URL = credentials('SONAR_HOST_URL')
        SLACK_TOKEN    = credentials('Slack-Token')
    }

    tools {
        maven 'localMaven'
        jdk 'localJdk'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: '', description: 'Optional: Git branch to build. Leave empty to detect automatically')
    }

    stages {
        stage('Init') {
            steps {
                script {
                    // ---------------- Helper Function: Maven ----------------
                    runMaven = { mavenGoal ->
                        withCredentials([
                            usernamePassword(credentialsId: 'Nexus-Credential', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS'),
                            string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')
                        ]) {
                            configFileProvider([configFile(fileId: 'maven-settings-template', variable: 'MAVEN_SETTINGS')]) {
                                script {
                                    // Create a temporary settings.xml that injects Nexus credentials
                                    def tmpSettings = "${env.WORKSPACE}/tmp-settings.xml"
                                    writeFile file: tmpSettings, text: readFile(MAVEN_SETTINGS)
                                    sh """
                                        sed -i 's|\\\${username}|$NEXUS_USER|g' $tmpSettings
                                        sed -i 's|\\\${password}|$NEXUS_PASS|g' $tmpSettings
                                        sed -i 's|\\\${nexus_private_ip}|${NEXUS_URL}|g' $tmpSettings
                                        mvn ${mavenGoal} --settings $tmpSettings
                                    """
                                    sh "rm -f $tmpSettings"
                                }
                            }
                        }
                    }

                    // ---------------- Helper Function: Ansible ----------------
                    deployAnsible = { hosts ->
                        withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
                            sh """
                                ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars "ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$hosts workspace_path=$WORKSPACE"
                            """
                        }
                    }
                }
            }
        }

        stage('Checkout') {
            steps {
                script {
                    def branchToBuild = params.BRANCH_NAME?.trim() ?: env.BRANCH_NAME ?: 'main'
                    echo "Building branch: ${branchToBuild}"
                    withCredentials([usernamePassword(credentialsId: 'Git-Credential', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        git branch: branchToBuild, url: GIT_REPO, credentialsId: 'Git-Credential'
                    }
                }
            }
        }

        stage('Build') { steps { script { runMaven('clean package') } } post { success { archiveArtifacts artifacts: '**/*.war' } } }
        stage('Unit Test') { steps { script { runMaven('test') } } }
        stage('Integration Test') { steps { script { runMaven('verify -DskipUnitTests') } } }
        stage('Checkstyle Analysis') { steps { script { runMaven('checkstyle:checkstyle') } } }

        stage('SonarQube Inspection') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')]) {
                        runMaven("sonar:sonar -Dsonar.projectKey=Java-WebApp-Project -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=$SONAR_TOKEN")
                    }
                }
            }
        }

        stage('SonarQube GateKeeper') { steps { timeout(time: 1, unit: 'HOURS') { waitForQualityGate abortPipeline: true } } }

        stage('Nexus Artifact Upload') {
            steps {
                script { 
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: NEXUS_URL,
                        groupId: 'webapp',
                        version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                        repository: 'maven-project-releases',
                        credentialsId: 'Nexus-Credential',
                        artifacts: [[artifactId: 'webapp', classifier: '', file: "${WORKSPACE}/webapp/target/webapp.war", type: 'war']]
                    )
                }
            }
        }

        stage('Deploy to Development') { steps { script { deployAnsible('dev') } } }
        stage('Deploy to Staging') { steps { script { deployAnsible('stage') } } }
        stage('QA Approval') { steps { input('Proceed to Production?') } }
        stage('Deploy to Production') { steps { script { deployAnsible('prod') } } }
    }

    post {
        always {
            slackSend channel: '#af-cicd-pipeline-2',
                      color: COLOR_MAP[currentBuild.currentResult],
                      tokenCredentialId: 'Slack-Token',
                      message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' Build #${env.BUILD_NUMBER}\nWorkspace: ${env.WORKSPACE}\nMore info: ${env.BUILD_URL}"
        }
    }
}
