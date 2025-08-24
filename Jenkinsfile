// ---------- Visual status colors for Slack ----------
def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

// ---------- Helper: Maven with Nexus creds + settings.xml templating + Sonar token ----------
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
                mvn ${mavenGoal} --settings $TMP_SETTINGS
                """
            } finally {
                sh "rm -f $TMP_SETTINGS"
            }
        }
    }
}

// ---------- Helper: Ansible deploy using Jenkins credential ----------
def deployAnsible = { envName ->
    withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
        sh """
        ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml \
          --extra-vars "ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_${envName} workspace_path=$WORKSPACE"
        """
    }
}

pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '25', artifactNumToKeepStr: '15'))
    }

    environment {
        WORKSPACE      = "${env.WORKSPACE}"
        GIT_REPO       = 'https://github.com/Oluwole-Faluwoye/realworld-cicd-pipeline-project.git'
        NEXUS_URL      = 'http://172.31.14.247:8081'
        SLACK_CHANNEL  = '#af-cicd-pipeline-2'
        SONAR_HOST_URL = 'http://172.31.6.142:9000'
        BRANCH_NAME    = 'main'  // default branch
        VERSION_TAG    = ''
    }

    tools {
        maven 'localMaven'
        jdk   'localJdk'
    }

    triggers {
        githubPush()  // Automatically trigger build on GitHub push
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    env.VERSION_TAG = "${env.BUILD_NUMBER}-${new Date().format('yyyyMMddHHmmss', TimeZone.getTimeZone('UTC'))}"
                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "Version tag: ${env.VERSION_TAG}"
                }
            }
        }

        stage('Checkout') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${env.BRANCH_NAME}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [
                            [$class: 'PruneStaleBranch'],
                            [$class: 'CleanBeforeCheckout'],
                            [$class: 'CloneOption', depth: 0, noTags: false, shallow: false, reference: '']
                        ],
                        userRemoteConfigs: [[
                            url: env.GIT_REPO,
                            credentialsId: 'Git-Credential',
                            refspec: '+refs/heads/*:refs/remotes/origin/*'
                        ]]
                    ])
                }
            }
        }

        stage('Build') {
            steps { script { runMaven('clean package') } }
            post { success { archiveArtifacts artifacts: '**/*.war', fingerprint: true } }
        }

        stage('Unit Test') { steps { script { runMaven('test') } } 
            post { always { junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true } } 
        }

        stage('Integration Test') { steps { script { runMaven('verify -DskipUnitTests') } } 
            post { always { junit testResults: '**/target/failsafe-reports/*.xml', allowEmptyResults: true } } 
        }

        stage('Checkstyle Analysis') { steps { script { runMaven('checkstyle:checkstyle') } } 
            post { always { archiveArtifacts artifacts: '**/target/checkstyle-result.xml', allowEmptyArchive: true } } 
        }

        stage('SonarQube Inspection') { steps { script { runMaven("sonar:sonar -Dsonar.projectKey=Java-WebApp-Project -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_TOKEN}") } } }

        stage('SonarQube GateKeeper') { steps { timeout(time: 1, unit: 'HOURS') { waitForQualityGate abortPipeline: true } } }

        stage('Nexus Artifact Upload') {
            steps {
                script {
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: NEXUS_URL,
                        groupId: 'webapp',
                        version: env.VERSION_TAG,
                        repository: 'maven-project-releases',
                        credentialsId: 'Nexus-Credential',
                        artifacts: [[artifactId: 'webapp', classifier: '', file: "${WORKSPACE}/webapp/target/webapp.war", type: 'war']]
                    )
                }
            }
        }

        stage('Deploy to Development') { steps { script { deployAnsible('dev') } } }
        stage('Deploy to Staging')     { steps { script { deployAnsible('stage') } } }
        stage('QA Approval')           { steps { input message: 'Proceed to Production?', ok: 'Deploy' } }
        stage('Deploy to Production')  { steps { script { deployAnsible('prod') } } }
    }

    post {
        always {
            script {
                withCredentials([string(credentialsId: 'Slack-Token', variable: 'SLACK_TOKEN')]) {
                    slackSend channel: SLACK_CHANNEL,
                              color: COLOR_MAP[currentBuild.currentResult],
                              tokenCredentialId: 'Slack-Token',
                              message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' Build #${env.BUILD_NUMBER}\nBranch: ${env.BRANCH_NAME}\nVersion: ${env.VERSION_TAG}\nWorkspace: ${WORKSPACE}\nMore info: ${env.BUILD_URL}"
                }
            }
        }
    }
}
