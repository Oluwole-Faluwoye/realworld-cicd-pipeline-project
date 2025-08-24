// ---------- Visual status colors for Slack ----------
def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

// ---------- Helper: Maven with Nexus creds + settings.xml templating ----------
def runMaven = { mavenGoal, useSonarJava11=false ->
    withCredentials([
        usernamePassword(credentialsId: 'Nexus-Credential', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS'),
        string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')
    ]) {
        configFileProvider([configFile(fileId: 'maven-settings-template', variable: 'MAVEN_SETTINGS')]) {
            def TMP_SETTINGS = "${env.WORKSPACE}/tmp-settings.xml"
            try {
                def envVars = [
                    "NEXUS_USER=$NEXUS_USER",
                    "NEXUS_PASS=$NEXUS_PASS"
                ]
                if (useSonarJava11) {
                    envVars += ["JAVA_HOME=${tool 'jdk11'}", "PATH=${tool 'jdk11'}/bin:${env.PATH}"]
                }
                withEnv(envVars) {
                    sh """
                        cp \$MAVEN_SETTINGS $TMP_SETTINGS
                        sed -i 's|\\\\\${username}|$NEXUS_USER|g' $TMP_SETTINGS
                        sed -i 's|\\\\\${password}|$NEXUS_PASS|g' $TMP_SETTINGS
                        sed -i 's|\\\\\${nexus_private_ip}|$NEXUS_URL|g' $TMP_SETTINGS
                        mvn ${mavenGoal} --settings $TMP_SETTINGS ${useSonarJava11 ? "-Dsonar.login=\\\$SONAR_TOKEN" : ""}
                    """
                }
            } finally {
                sh "rm -f $TMP_SETTINGS"
            }
        }
    }
}

// ---------- Helper: Ansible deploy using Jenkins credential ----------
def deployAnsible = { envName ->
    withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
        withEnv(["ANSIBLE_USER=$USER_NAME", "ANSIBLE_PASS=$PASSWORD"]) {
            sh """
                ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml \
                  --extra-vars "ansible_user=\\\$ANSIBLE_USER ansible_password=\\\$ANSIBLE_PASS hosts=tag_Environment_${envName} workspace_path=$WORKSPACE"
            """
        }
    }
}

pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '25', artifactNumToKeepStr: '15'))
    }

    environment {
        WORKSPACE      = "${env.WORKSPACE}"
        GIT_REPO       = 'https://github.com/Oluwole-Faluwoye/realworld-cicd-pipeline-project.git'
        NEXUS_URL      = 'http://172.31.14.247:8081'
        SLACK_CHANNEL  = '#af-cicd-pipeline-2'
        SONAR_HOST_URL = 'http://172.31.6.142:9000'
        DEFAULT_BRANCH = 'main'
        VERSION_TAG    = ''
    }

    tools {
        maven 'localMaven'
        jdk   'localJdk' // Java 17
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Pipeline Start') {
            steps { ansiColor('xterm') { echo ">>> Pipeline starting <<<" } }
        }

        stage('Prepare') {
            steps {
                ansiColor('xterm') {
                    script {
                        env.BRANCH_NAME = env.BRANCH_NAME ?: env.DEFAULT_BRANCH
                        env.VERSION_TAG = "${env.BUILD_NUMBER}-${new Date().format('yyyyMMddHHmmss', TimeZone.getTimeZone('UTC'))}"
                        echo "Branch: ${env.BRANCH_NAME}"
                        echo "Version tag: ${env.VERSION_TAG}"
                    }
                }
            }
        }

        stage('Checkout') {
            steps {
                ansiColor('xterm') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${env.BRANCH_NAME}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [
                            [$class: 'PruneStaleBranch'],
                            [$class: 'CleanBeforeCheckout']
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
            steps { ansiColor('xterm') { script { runMaven('clean package') } } }
            post { success { archiveArtifacts artifacts: '**/*.war', fingerprint: true } }
        }

        stage('Unit Test') {
            steps { ansiColor('xterm') { script { runMaven('test') } } }
            post { always { junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true } }
        }

        stage('Integration Test') {
            steps { ansiColor('xterm') { script { runMaven('verify -DskipUnitTests') } } }
            post { always { junit testResults: '**/target/failsafe-reports/*.xml', allowEmptyResults: true } }
        }

        stage('Checkstyle Analysis') {
            steps { ansiColor('xterm') { script { runMaven('checkstyle:checkstyle') } } }
            post { always { archiveArtifacts artifacts: '**/target/checkstyle-result.xml', allowEmptyArchive: true } }
        }

        stage('SonarQube Inspection') {
            steps {
                ansiColor('xterm') {
                    script {
                        // Uses Java 11 only for Sonar
                        runMaven("sonar:sonar -Dsonar.projectKey=Java-WebApp-Project -Dsonar.host.url=${SONAR_HOST_URL}", true)
                    }
                }
            }
        }

        stage('SonarQube GateKeeper') {
            steps { timeout(time: 1, unit: 'HOURS') { waitForQualityGate abortPipeline: true } }
        }

        stage('Nexus Artifact Upload') {
            steps {
                ansiColor('xterm') {
                    script {
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: NEXUS_URL,
                            groupId: 'webapp',
                            version: env.VERSION_TAG,
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
        }

        stage('Deploy to Development') { steps { script { deployAnsible('dev') } } }
        stage('Deploy to Staging') { steps { script { deployAnsible('stage') } } }
        stage('QA Approval') { steps { input message: 'Proceed to Production?', ok: 'Deploy' } }
        stage('Deploy to Production') { steps { script { deployAnsible('prod') } } }
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
