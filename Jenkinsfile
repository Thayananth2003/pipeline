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
                nexusArtifactUploader artifacts: [[artifactId: 'cdro-maven', classifier: '', file: '/var/lib/jenkins/workspace/Intenship/Thaya/CDRO-Maven/target/demo-0.0.1-SNAPSHOT.jar', type: 'jar']], credentialsId: 'nexus31', groupId: 'com.logicfocus', nexusUrl: '192.168.1.163:32007', nexusVersion: 'nexus3', protocol: 'http', repository: 'cdro-maven', version: '1.0.1'
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
        stage('Build-Data artifactory') {
            steps{
                archiveArtifacts artifacts: 'BuildData.json', followSymlinks: false
            }
        }
        stage('Trigger CDRO') {
            steps{
                cloudBeesFlowRunPipeline addParam: '{"pipeline":{"pipelineName":"CDRO-Maven","parameters":[]}}', configuration: '/project/Default/pluginConfiguration/jenkins', pipelineName: 'CDRO-Maven', projectName: 'Naveen', stageOption: 'runAllStages', stagesToRun: '{"pipeline":{"pipelineName":"CDRO-Maven","stages":[{"stageName":"Build Report","stageValue":""},{"stageName":"Application Process","stageValue":""},{"stageName":"REPORT","stageValue":""},{"stageName":"SIT","stageValue":""},{"stageName":"UAT","stageValue":""}]}}', startingStage: ''
            }
        }
        stage('SAST Report') {
            environment {
                Project_key = "cdro-maven"
                Sonar_URL = "http://192.168.1.163:32003"
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'sonar_1', variable: 'SONAR_TOKEN')]) {
                        try {
                            def sonarReport = sh(script: """
                                curl -s -u ${SONAR_TOKEN}: '${Sonar_URL}/api/issues/search?componentKeys=${Project_Key}&types=BUG,VULNERABILITY,CODE_SMELL&ps=500' | jq -r '[.issues[] | {severity}] | group_by(.severity) | map({(.[0].severity): length}) | add'
                            """, returnStdout: true).trim()
                            writeFile file: 'SAST_Report.json', text: sonarReport

                        }
                        catch (Exception e) {
                            error "Failed to retrieve SonarQube report: ${e.message}"
                        }
                    }
                }
            }
        }
        stage('Upload SAST Report') {
            steps {
                archiveArtifacts artifacts: 'SAST_Report.json', followSymlinks: false
            }
        }
    }
}

    
