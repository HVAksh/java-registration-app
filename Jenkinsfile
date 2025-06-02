pipeline {
    agent any
    tools {
        maven 'maven'
    }

    parameters{
        choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
        string(name: 'DockerHubUser', description: 'name of the docker user', defaultValue: 'hvaksh')
    }
    environment {
        APP_NAME = "java-registration-app"
        RELEASE = "1.0.0"
        DOCKER_PASS = 'docker'
        IMAGE_NAME = "${params.DockerHubUser}"+"/"+"${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }
    stages{

        stage('Clean Workspace') {
                    when {expression {params.action == 'create'}}
                steps{
                    cleanWs()
            }
        }
        stage('Checkout from Git') {
                    when {expression {params.action == 'create'}}
            steps{
                git branch: 'main', url: 'https://github.com/HVAksh/java-registration-app.git'
            }
        }
        stage('build, unit test and Integration test') {
                    when {expression {params.action == 'create'}}
            steps{
                dir('webapp'){
                    sh 'mvn package'
                }
            }
        }
        stage('Static Code Analysis: SonarQube') {
                    when {expression {params.action == 'create'}}
            steps{
                withSonarQubeEnv('SonarQube-Server') {
                    dir('webapp'){
                        sh 'mvn -U clean install sonar:sonar'
                    }
                }
            }
        }
        stage('QualityGate Status Check') {
                    when {expression {params.action == 'create'}}
            steps{
                script {
                    waitForQualityGate abortPipeline:  false, credentialsId: 'SonarQube-Token'
                }
            }
        }
        stage('Artifactory configuration') {
                    when {expression {params.action == 'create'}}
            steps{
                rtServer (
                    id: "jfrog-server",
                    url: "http://192.168.147.128:8082/artifactory",
                    credentialsId: "jfrog"
                )
                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release-local",
                    snapshotRepo: "libs-snapshot-local"
                )
                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release",
                    snapshotRepo: "libs-snapshot"
                )
            }
        }
        stage('Deploy Artifacts') {
                    when {expression {params.action == 'create'}}
            steps{
                rtMavenRun (
                    tool: "maven",
                    pom: 'webapp/pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
            }
        }
        stage('Build Publish info') {
                    when {expression {params.action == 'create'}}
            steps{
                rtPublishBuildInfo (
                    serverId: "jfrog-server"
                )
            }
        }
        stage('Trivy FS scan') {
                    when {expression {params.action == 'create'}}
            steps{
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage('Build and Publish Docker Image') {
                    when {expression {params.action == 'create'}}
            steps{
                script {
                    
                    docker.withRegistry ('', "${DOCKER_PASS}") {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }
                    docker.withRegistry ('', "${DOCKER_PASS}") {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push("latest")
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
                    when {expression {params.action == 'create'}}
            steps{
                script {
                    sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image hvaksh/java-registration-app:latest --timeout 60m --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table > trivyimage.txt')
                }
            }
        }
        stage('Clean Artifacts') {
                    when {expression {params.action == 'create'}}
            steps{
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }
    }
}