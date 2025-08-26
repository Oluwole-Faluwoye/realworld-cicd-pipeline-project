def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
    'UNSTABLE': 'warning'
]

pipeline {
    agent any

    environment {
        WORKSPACE = "${env.WORKSPACE}"
        NEXUS_CREDENTIAL_ID = 'Nexus-Credential'
    }

    tools {
        maven 'localMaven'
        jdk 'localJdk'
    }

    stages {

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
            post {
                success {
                    echo 'Archiving artifact...'
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Integration Test') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('Checkstyle Code Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Checkstyle report.'
                }
            }
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
                            withCredentials([string(credentialsId: 'Sonarqube-Token', variable: 'SONAR_TOKEN')]) {
                                sh """
                                mvn sonar:sonar \
                                -Dsonar.projectKey=Java-WebApp-Project \
                                -Dsonar.host.url=http://172.31.6.182:9000 \
                                -Dsonar.login=${SONAR_TOKEN}
                                """
                            }
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

        stage('Nexus Artifact Uploader') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '172.31.1.49:8081',
                    groupId: 'webapp',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'maven-project-releases',
                    credentialsId: "${NEXUS_CREDENTIAL_ID}",
                    artifacts: [
                        [artifactId: 'webapp',
                         classifier: '',
                         file: "${WORKSPACE}/webapp/target/webapp.war",
                         type: 'war']
                    ]
                )
            }
        }

        // ---- DEPLOY STAGES USING PASSWORD-BASED SSH ----

        stage('Deploy to Development Env') {
            environment { HOSTS = 'dev' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
                    sh """
                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml \
                    --extra-vars "ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE" \
                    --ask-pass
                    """
                }
            }
        }

        stage('Deploy to Staging Env') {
            environment { HOSTS = 'stage' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
                    sh """
                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml \
                    --extra-vars "ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE" \
                    --ask-pass
                    """
                }
            }
        }

        stage('Quality Assurance Approval') {
            steps {
                input('Do you want to proceed to Production?')
            }
        }

        stage('Deploy to Production Env') {
            environment { HOSTS = 'prod' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', usernameVariable: 'USER_NAME', passwordVariable: 'PASSWORD')]) {
                    sh """
                    ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml \
                    --extra-vars "ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE" \
                    --ask-pass
                    """
                }
            }
        }
    }

    post {
        always {
            echo 'Sending Slack notifications...'
            slackSend(
                channel: '#af-cicd-pipeline-2',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job Name '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \nBuild Timestamp: ${env.BUILD_TIMESTAMP} \nProject Workspace: ${env.WORKSPACE} \nMore info at: ${env.BUILD_URL}"
            )
        }
    }
}
