def mainDir="Part2_Docker/Chapter06/3-1_nexus-jenkins-docker"
def region="ap-northeast-2"
def nexusUrl="<Nexus VM의 프라이빗 IP DNS 이름>:5443"
def repository="container-registry"
def deployHost="<Deploy VM의 프라이빗 IP DNS 이름>"
def tagName="1.0"
def nexusid="<Nexus 접근 계정명>"
def nexuspw="<Nexus 접근 비밀번호>"
def jenkins_ip="<Jenkins VM의 프라이빗 IP DNS 이름>"

pipeline {
    agent any

    stages {
        stage('Pull Codes from Github'){
            steps{
                checkout scm
            }
        }
        stage('Build Codes by Gradle') {
            steps {
                sh """
                cd ${mainDir}
                ./gradlew clean build
                """
            }
        }
        stage('Build Docker Image by Jib & Push to Nexus Registry') {
            steps {
                sh """
                    cd ${mainDir}
                    docker login -u ${nexusid} -p ${nexuspw} ${nexusUrl}
                    ./gradlew jib -Djib.to.image=${nexusUrl}/${repository}:${tagName} -Djib.console='plain'
                """
            }
        }
        stage('Scan Security CVE at Clair Scanner') {
            steps {
                script {
                    try {
                        clair_ip = sh(script: "docker inspect -f '{{ .NetworkSettings.IPAddress }}' clair", returnStdout: true).trim()
                        sh """
                            docker pull ${nexusUrl}/${repository}:${tagName}
                            clair-scanner --ip ${jenkins_ip} --clair='http://${clair_ip}:6060' --log='clair.log' --report='report.txt' ${nexusUrl}/${repository}:${tagName}
                        """
                    } catch (err) {
                        echo err.getMessage()
                    }
                }
                echo currentBuild.result
            }
        }
        stage('Deploy container to AWS EC2 VM'){
            steps{
                sshagent(credentials : ["deploy-key"]) {
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@${deployHost} \
                     'docker login -u ${nexusid} -p ${nexuspw} ${nexusUrl}; \
                      docker run -d -p 80:8080 -t ${nexusUrl}/${repository}:${tagName};'"
                }
            }
        }
    }
}
