def notifyBuild(String buildStatus = 'STARTED') {
    // Build status of null means success.
    buildStatus =  buildStatus ?: 'SUCCESS'
    if (buildStatus == 'STARTED') {
        colorCode = '#FFFF00'
    } else if (buildStatus == 'SUCCESS') {
        colorCode = '#00FF00'
    } else {
        colorCode = '#FF0000'
    }
    // Send notification.
    def msg = "${buildStatus}: `${env.JOB_NAME}` #${env.BUILD_NUMBER}:\n${env.BUILD_URL}"
    slackSend(color: colorCode, message: msg)
}
node('master'){
    try {
        notifyBuild('STARTED')
        stage('1. Checking Out Code'){
            checkout([$class: 'GitSCM', branches: [[name: '*/dev']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'Git-Creds', url: 'https://github.com/berezovsky15/hello-world-war.git']]])
        }
        stage('2. Building'){
            sh label: '', script: "mvn clean package"
        }
        stage('3. Analyzing'){
            sh label: '', script: '''cd /opt/tomcat/.jenkins/workspace/M6-hello-world-war
                                      mvn verify sonar:sonar'''
        }
        stage('4. Building Dockerfile'){
            sh label: '', script: '''cp /opt/tomcat/.jenkins/workspace/M6-hello-world-war /opt/tomcat/.jenkins/workspace/M6-hello-world-war
                                     docker build .'''
        }
        stage('5. Tagging Docker Image'){
            sh label: '', script: 'docker tag $(docker images | grep \'<none>\' | head -n 1 | awk \'{print $3}\') java-app:${BUILD_ID}'
        }
        stage('6. Uploading Image To Nexus'){
            withCredentials([usernamePassword(credentialsId: 'Nexus-Docker', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                sh label: '', script: '''docker login -u $USERNAME -p $PASSWORD 192.168.1.233:8082
                                         docker tag java-app:${BUILD_ID} 192.168.1.233:8082/java-app:${BUILD_ID}
                                         docker push 192.168.1.233:8082/java-app:${BUILD_ID}
                                         docker rmi $(docker images --filter=reference="192.168.1.233:8082/java-app*" -q) -f
                                         docker logout 192.168.1.233'''
            }
        }
    } catch (e) {
    // If there was an exception thrown, the build failed.
        currentBuild.result = "FAILED"
        throw e
    } finally {
    // Success or failure, always send notification.
        stage('7. Notifying Slack'){
            notifyBuild(currentBuild.result)
        }
    }
}
  
