def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]

pipeline {
    agent any

    environment {
        WORKSPACE      = "${env.WORKSPACE}"
        NEXUS_URL      = credentials('NEXUS_URL')      // Nexus server URL
        SONAR_HOST_URL = credentials('SONAR_HOST_URL') // SonarQube server URL
        GIT_REPO       = 'https://github.com/Oluwole-Faluwoye/realworld-cicd-pipeline-project.git'
    }

    tools {
        maven 'localMaven'
        jdk 'localJdk'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: '', description: 'Optional: Git branch to build')
    }

    stages {

        stage('Init') {
            steps {
                script {
                    // ---------------- Helper Function: Maven ----------------
                    def runMaven = { mavenGoal, nexusUrl ->
                        withCredentials([
                            usernamePassword(credentialsId: 'Nexus-Credential', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS'),
                            string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')
                        ]) {
                            configFileProvider([configFile(fileId: 'maven-settings-template', variable: 'MAVEN_SETTINGS')]) {
                                def TMP_SETTINGS = "${env.WORKSPACE}/tmp-settings.xml"
                                try {
                                    sh """
                                    cp \$MAVEN_SETTINGS $TMP_SETTINGS
                                    sed -i 's|\\\${username}|$NEXUS_USER|g' $TMP_SETTINGS
                                    sed -i 's|\\\${password}|$NEXUS_PASS|g' $TMP_SETTINGS
                                    sed -i 's|\\\${nexus_private_ip}|$nexusUrl|g' $TMP_SETTINGS
                                    mvn $mavenGoal --settings $TMP_SETTINGS
                                    """
                                } finally { sh "rm -f $TMP_SETTINGS" }
                            }
                        }
                    }

                    // ---------------- Helper Function: Ansible ----------------
                    def deployAnsible = { hosts ->
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

                    // ---------------- Conditional Git Clone ----------------
                    if (Jenkins.instance.getCredential('Git-Credential')) {
                        // Private repo with credentials
                        withCredentials([usernamePassword(credentialsId: 'Git-Credential', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                            git branch: branchToBuild, url: GIT_REPO, credentialsId: 'Git-Credential'
                        }
                    } else {
                        // Public repo, no credentials
                        git branch: branchToBuild, url: GIT_REPO
                    }
                }
            }
        }

        stage('Build') {
            steps { script { runMaven('clean package', NEXUS_URL) } }
            post { success { archiveArtifacts artifacts: '**/*.war' } }
        }

        stage('Unit Test') { steps { script { runMaven('test', NEXUS_URL) } } }
        stage('Integration Test') { steps { script { runMaven('verify -DskipUnitTests', NEXUS_URL) } } }
        stage('Checkstyle Analysis') { steps { script { runMaven('checkstyle:checkstyle', NEXUS_URL) } } }

        stage('SonarQube Inspection') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')]) {
                        runMaven("sonar:sonar -Dsonar.projectKey=Java-WebApp-Project -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=$SONAR_TOKEN", NEXUS_URL)
                    }
                }
            }
        }

        stage('SonarQube GateKeeper') { steps { timeout(time: 1, unit: 'HOURS') { waitForQualityGate abortPipeline: true } } }

        stage('Nexus Artifact Upload') {
            steps {
                script { 
                    withCredentials([usernamePassword(credentialsId: 'Nexus-Credential', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
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
