def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

// ---------------------- Helper Functions (Global) ----------------------
def runMaven = { mavenGoal ->
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
                sed -i 's|\\\${nexus_private_ip}|$NEXUS_URL|g' $TMP_SETTINGS
                mvn $mavenGoal --settings $TMP_SETTINGS
                """
            } finally {
                sh "rm -f $TMP_SETTINGS"
            }
        }
    }
}

def deployAnsible = { envName ->
    withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
        sh """
        ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml \
        --extra-vars "ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$envName workspace_path=$WORKSPACE"
        """
    }
}

// ---------------------- Pipeline ----------------------
pipeline {
    agent any

    environment {
        WORKSPACE      = "${env.WORKSPACE}"
        GIT_REPO       = 'https://github.com/Oluwole-Faluwoye/realworld-cicd-pipeline-project.git'
        NEXUS_URL      = 'http://172.31.14.247:8081'
        SLACK_CHANNEL  = '#af-cicd-pipeline-2'
        SONAR_HOST_URL = 'http://172.31.6.142:9000'
        DEFAULT_BRANCH = 'main'
    }

    tools {
        maven 'localMaven'
        jdk 'localJdk'
    }

    parameters {
        // Active Choices Dropdown for Branches
        [$class: 'CascadeChoiceParameter',
         choiceType: 'PT_SINGLE_SELECT',
         description: 'Select branch to build (manual build only)',
         name: 'BRANCH_NAME',
         randomName: 'choice-parameter-123456',
         script: [
            $class: 'GroovyScript',
            fallbackScript: [classpath: [], sandbox: false, script: 'return ["${DEFAULT_BRANCH}"]'],
            script: [classpath: [], sandbox: false, script: '''
                try {
                    def proc = "git ls-remote --heads ${GIT_REPO}".execute()
                    proc.waitFor()
                    return proc.in.text.readLines().collect { it.split()[1].replace("refs/heads/", "") }
                } catch (err) {
                    return ["${DEFAULT_BRANCH}"]
                }
            ''']
         ]
        ]

        booleanParam(name: 'USE_GIT_CREDENTIAL', defaultValue: false, description: 'Enable if using private Git repo')
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    def branchToBuild = params.BRANCH_NAME?.trim()
                    if (!branchToBuild) {
                        branchToBuild = env.GIT_BRANCH?.replaceFirst(/^origin\//, '') ?: env.DEFAULT_BRANCH
                    }
                    echo "Building branch: ${branchToBuild}"

                    if (params.USE_GIT_CREDENTIAL) {
                        git branch: branchToBuild, url: GIT_REPO, credentialsId: 'Git-Credential'
                    } else {
                        git branch: branchToBuild, url: GIT_REPO
                    }
                }
            }
        }

        stage('Build') {
            steps { script { runMaven('clean package') } }
            post { success { archiveArtifacts artifacts: '**/*.war' } }
        }

        stage('Unit Test') { steps { script { runMaven('test') } } }
        stage('Integration Test') { steps { script { runMaven('verify -DskipUnitTests') } } }
        stage('Checkstyle Analysis') { steps { script { runMaven('checkstyle:checkstyle') } } }

        stage('SonarQube Inspection') {
            steps { script { runMaven("sonar:sonar -Dsonar.projectKey=Java-WebApp-Project -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_TOKEN}") } }
        }

        stage('SonarQube GateKeeper') {
            steps { timeout(time: 1, unit: 'HOURS') { waitForQualityGate abortPipeline: true } }
        }

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
        stage('Deploy to Staging')     { steps { script { deployAnsible('stage') } } }
        stage('QA Approval')           { steps { input('Proceed to Production?') } }
        stage('Deploy to Production')  { steps { script { deployAnsible('prod') } } }

    }

    post {
        always {
            script {
                withCredentials([string(credentialsId: 'Slack-Token', variable: 'SLACK_TOKEN')]) {
                    slackSend channel: env.SLACK_CHANNEL,
                              color: COLOR_MAP[currentBuild.currentResult],
                              tokenCredentialId: 'Slack-Token',
                              message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' Build #${env.BUILD_NUMBER}\nWorkspace: ${env.WORKSPACE}\nMore info: ${env.BUILD_URL}"
                }
            }
        }
    }
}
