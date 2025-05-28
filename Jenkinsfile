pipeline{
    agent any
    tools{
        maven 'maven'
    }
    environment {
        APP_NAME = "jave-registration-app"
        RELEASE = "1.0.0"
        DOCKER_USER = "hvaksh"
        IMAGE_NAME = "${DOCKER_USER}"+"/"+"${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }
    stages{
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/HVAksh/java-registration-app.git'
            }
        }
        stage('unit test') {
            steps {
                script {
                    sh 'mvn test'
                }
            }
        }
        stage('integration test') {
            steps {
                script {
                    sh 'mvn verify -Pintegration-tests -DskipTests=true'
                }
            }
        }
        stage('static code analysis: sonarqube') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                dir('webapp'){
                sh 'mvn -U clean install sonar:sonar'
                }
              }
            }
        }
        stage('quality gate status check: sonarqube') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube-token'
                }
            }
        }
        stage('mavenbuild') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }
        stage('artifactory configuration') {
            steps {
                rtServer (
                    id: "jfrog"
                    url: "http://localhost:8082/artifactory"
                    credentialsId: "jfrog"
                )
                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog",
                    releaseRepo: "libs-release-local",
                    snapshotRepo: "libs-snapshot-local"
                )
                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog",
                    releaseRepo: "libs-release",
                    snapshotRepo: "libs-snapshot"
                )
            }
        }
        stage('Deploy artifacts') {
            steps {
                rtMavenRun (
                    tool: "maven", 
                    pom: 'webapp/pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
            }
        }
        stage('build publish info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "jfrog"
                )
            }
        }
        stage('trivy FS scan') {
            steps {
                command
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                command
            }
        }
        stage('Trivy Image Scan') {
            steps {
                command
            }
        }
        stage('CleanUP artifacts') {
            steps {
                command
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                command
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'akshchaudhary92@gmail.com',                              
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}