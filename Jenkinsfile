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
                sh """#!/bin/bash
                    cp "\$MAVEN_SETTINGS" "$TMP_SETTINGS"
                    sed -i "s|\\\${NEXUS_USER}|$NEXUS_USER|g" "$TMP_SETTINGS"
                    sed -i "s|\\\${NEXUS_PASS}|$NEXUS_PASS|g" "$TMP_SETTINGS"
                    sed -i "s|\\\${NEXUS_URL}|$NEXUS_URL|g" "$TMP_SETTINGS"
                    sed -i "s|\\\${SONAR_TOKEN}|$SONAR_TOKEN|g" "$TMP_SETTINGS"
                    mvn ${mavenGoal} --settings "$TMP_SETTINGS"
                """
            } finally {
                sh "rm -f $TMP_SETTINGS"
            }
        }
    }
}

// ---------- Helper: Ansible deploy using Jenkins credentials (secure & AWS-ready) ----------
def deployAnsible = { envName ->
    withCredentials([
        usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'ANSIBLE_USER', passwordVariable: 'ANSIBLE_PASS'),
        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
    ]) {
        sh(script: """
            #!/bin/bash
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

            ansible-playbook -i "${WORKSPACE}/ansible-config/aws_ec2.yaml" "${WORKSPACE}/deploy.yaml" \\
              --extra-vars "ansible_user=$ANSIBLE_USER ansible_password=$ANSIBLE_PASS hosts=tag_Environment_${envName} workspace_path=$WORKSPACE"
        """, returnStatus: false)
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
        NEXUS_URL      = '172.31.2.149:8081'
        SLACK_CHANNEL  = '#af-cicd-pipeline-2'
        SONAR_HOST_URL = 'http://172.31.8.156:9000'
        DEFAULT_BRANCH = 'main'
    }

    tools {
        maven 'localMaven'
        jdk   'localJdk'
    }

    triggers { githubPush() }

    stages {
        stage('Pipeline Start') {
            steps { ansiColor('xterm') { echo ">>> Pipeline starting <<<" } }
        }

        stage('Prepare') {
            steps {
                ansiColor('xterm') {
                    script {
                        env.BRANCH_NAME = env.BRANCH_NAME ?: env.DEFAULT_BRANCH
                        echo "Branch: ${env.BRANCH_NAME}"
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
                        extensions: [[$class: 'PruneStaleBranch'], [$class: 'CleanBeforeCheckout']],
                        userRemoteConfigs: [[url: env.GIT_REPO, credentialsId: 'Git-Credential', refspec: '+refs/heads/*:refs/remotes/origin/*']]
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
                script {
                    env.JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
                    env.PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
                    echo "Using JAVA_HOME = ${env.JAVA_HOME}"
                    sh 'java -version'

                    withEnv([
                        'MAVEN_OPTS=--add-opens java.base/java.lang=ALL-UNNAMED ' +
                                     '--add-opens java.base/java.io=ALL-UNNAMED ' +
                                     '--add-opens java.base/java.util=ALL-UNNAMED ' +
                                     '--add-opens java.base/java.lang.reflect=ALL-UNNAMED'
                    ]) {
                        withSonarQubeEnv('SonarQube') {
                            runMaven('sonar:sonar')
                        }
                    }
                }
            }
        }

        stage('SonarQube GateKeeper') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Nexus Artifact Upload') {
            steps {
                ansiColor('xterm') {
                    script {
                        echo "Deploying webapp to Nexus via Maven..."
                        dir('webapp') {
                            runMaven('deploy')
                        }
                    }
                }
            }
        }

        // ---------- Ansible Deploy Stages ----------
        stage('Deploy Environments') {
            steps {
                script {
                    def deployEnvironments = ['dev', 'stage', 'prod']
                    deployEnvironments.each { envName ->
                        if (envName == 'prod') {
                            input message: 'Proceed to Production?', ok: 'Deploy'
                        }
                        deployAnsible(envName)
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                withCredentials([string(credentialsId: 'Slack-Token', variable: 'SLACK_TOKEN')]) {
                    slackSend channel: SLACK_CHANNEL,
                              color: COLOR_MAP[currentBuild.currentResult],
                              tokenCredentialId: 'Slack-Token',
                              message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' Build #${env.BUILD_NUMBER}\nBranch: ${env.BRANCH_NAME}\nWorkspace: ${WORKSPACE}\nMore info: ${env.BUILD_URL}"
                }
            }
        }
    }
}
