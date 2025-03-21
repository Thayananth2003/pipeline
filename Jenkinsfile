pipeline {
    agent any

    stages {
        stage('REPO-Checkout') {
            steps {
                script {
                 checkout scmGit(branches: [[name: params.BRANCH]], extensions: [], userRemoteConfigs: [[credentialsId: 'Thaya', url: params.REPO_URL]])
               }
            }
        }
        stage('Maven build') {
            tools {
                jdk 'java-17'
            }
            steps {
                sh 'mvn package'
            }
        }
         stage('SAST-analysis') {
            tools {
                jdk 'java-17'
            }
            environment {
                scannerHome = tool 'SonarQubeScanner'
                projectName = "cdro-maven"
            }
            steps {
                withSonarQubeEnv('sonar_1') {
                    sh """
                        export JAVA_HOME=\$JAVA_HOME
                        export PATH=\$JAVA_HOME/bin:\$PATH
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=${projectName} \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=.
                    """
                }
            }
        }
        stage('publish in nexus') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'CDRO-Maven', classifier: '', file: '/var/lib/jenkins/workspace/Intenship/Thaya/CDRO-Maven/target/demo-0.0.1-SNAPSHOT.jar', type: 'jar']], credentialsId: 'nexus31', groupId: 'com.logicfocus', nexusUrl: '192.168.1.163:32007', nexusVersion: 'nexus3', protocol: 'http', repository: 'CDRO-Maven', version: '1.0.1'
            }
            
        }
        stage('Download artifactory') {
            steps {
                archiveArtifacts artifacts: 'target/demo-0.0.1-SNAPSHOT.jar', followSymlinks: false
            }    
        }
         stage('Generate builddata.json') {
            steps {
                script {
                    def currentPath = pwd()
                    def path = currentPath.split("/")
                    def folderName = path[-3]
                    def repoName = sh(script: 'basename `git config --get remote.origin.url` .git', returnStdout: true).trim()
                    def buildData = [
                        BuildNumber: env.BUILD_NUMBER,
                        jobName: env.JOB_NAME,
                        jobUrl: env.BUILD_URL,
                        "Repo-name" : "${repoName}",
                        gitCommit: sh(script: 'git rev-parse HEAD', returnStdout: true).trim(),
                        "Folder" : "${folderName}",
                        ]
                    writeFile file: 'BuildData.json' , text: groovy.json.JsonOutput.prettyPrint(groovy.json.JsonOutput.toJson(buildData))
                }
            }
        }
        stage('BuildData artifactory') {
            steps{
                archiveArtifacts artifacts: 'BuildData.json', followSymlinks: false
            }
        }
    }
}

    
