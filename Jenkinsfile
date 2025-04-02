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
    post {
            always {
                script {
                    def emailBody = """
                    <html>
                    <body>
                    <h2>Commit Details:</h2>
                    <table border="1" cellspacing="0" cellpadding="5">
                        <tr>
                            <td><b>REPOSITORY NAME</b></td>
                            <td>${env.REPO_NAME}</td>
                        </tr>
                        <tr>
                            <td><b>REPOSITORY URL</b></td>
                            <td>${env.REPO_URL}</td>
                        </tr>
                        <tr>
                            <td><b>COMMIT ID</b></td>
                            <td>${env.COMMIT_ID}</td>
                        </tr>
                        <tr>
                            <td><b>VERSION NUMBER</b></td>
                            <td>1.0.1</td>
                        </tr>
                        <tr>
                            <td><b>LAST COMMIT DATE</b></td>
                            <td>${env.LAST_COMMIT_DATE}</td>
                        </tr>
                        <tr>
                            <td><b>BRANCH NAME</b></td>
                            <td>release/r1.0</td>
                        </tr>
                    </table>
                    <br>
                    <h2>SonarQube Quality Gate:</h2>
                    <table border="1" cellspacing="0" cellpadding="5">
                        <tr>
                            <th colspan="3" style="background-color: #FFFFCC; text-align: left;">Quality Gate</th>
                        </tr>
                        <tr>
                            <td colspan="3" style="background-color: ${env.SONAR_QUALITY_GATE_STATUS == '0' ? 'green' : 'red'}; color: white; text-align: center;">
                                ${env.SONAR_QUALITY_GATE_STATUS == '0' ? 'Quality Gate Passed' : 'Quality Gate Failed'}
                            </td>
                        </tr>
                        <tr style="background-color: #BCE8F1;">
                            <th>COMPONENT</th>
                            <th>EXPECTED</th>
                            <th>ACTUAL</th>
                        </tr>
                        <tr>
                            <td>BLOCKER</td>
                            <td>0</td>
                            <td>${env.BLOCKER_COUNT}</td>
                        </tr>
                        <tr>
                            <td>CRITICAL</td>
                            <td>0</td>
                            <td>${env.CRITICAL_COUNT}</td>
                        </tr>
                        <tr>
                            <td>MAJOR</td>
                            <td>0</td>
                            <td>${env.MAJOR_COUNT}</td>
                        </tr>
                        <tr>
                            <td>MINOR</td>
                            <td>0</td>
                            <td>${env.MINOR_COUNT}</td>
                        </tr>
                        <tr>
                            <td>INFO</td>
                            <td>0</td>
                            <td>0</td>
                        </tr>
                    </table>
                    <p>CI Run has been completed successfully.</p>
                    """
                    emailext(
                        subject: "Build Notification/${env.REPO_NAME}",
                        body: emailBody,
                        to: 'naveen.s@logicfocus.net',
                        mimeType: 'text/html'
                    )
                }
            }
        }
}

    
