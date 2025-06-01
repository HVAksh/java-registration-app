pipeline {
    agent any

    parameters{
        choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
        string(name: 'DockerHubUser', description: 'name of the docker user', defaultValue: 'hvaksh')
    }
    environment {
        APP_NAME = "javapp"
        RELEASE = "1.0.0"
        IMAGE_NAME = "${params.DockerHubUser}"+"/"+"${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }
    stages{

        stage('Clean Workspace') {
                    when {expression {param.action == 'create'}}
            steps{
                cleanWs()
            }
        }
        stage('git checkout') {
                    when {expression {param.action == 'create'}}
            steps{
                git branch: 'main', url: 'https://github.com/HVAksh/java-registration-app.git'
            }
        }
        stage('build, unit test and Integration test') {
                    when {expression {param.action == 'create'}}
            steps{
                sh 'mvn clean verify -Pintegration-tests'
            }
        }
        stage('Static Code Analysis: SonarQube') {
                    when {expression {param.action == 'create'}}
            steps{
                withSonarQubeEnv('SonarQube-Server')
                dir('webapp')
                sh 'mvn sonar:sonar'
            }
        }
        stage('QualityGate Status Check') {
                    when {expression {param.action == 'create'}}
            steps{
                script {
                    waitForQualityGate abortPipeline:  false, credentialsId: 'SonarQube-Token'
                }
            }
        }
        stage('Artifactory configuration') {
                    when {expression {param.action == 'create'}}
            steps{
                rtServer (
                    id: "jfrog-server",
                    url: "http://192.168.147.128:8082/artifactory",
                    credentialsId: "jfrog"
                )
                rtMavenDeployer (
                    id: "Maven-Deployer",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release-local",
                    snapshotRepo: "libs-snapshot-local"
                )
                rtMavenResolver (
                    id: "Maven-Resolver",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release",
                    snapshotRepo: "libs-snapshot"
                )
            }
        }
        stage('Deploy Artifacts') {
                    when {expression {param.action == 'create'}}
            steps{
                rtMavenRun (
                    tool: "maven",
                    pom: "webapp/pom.xml",
                    goals: "clean install",
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
            }
        }
        stage('Build Publish info') {
                    when {expression {param.action == 'create'}}
            steps{
                rtPublishBuildInfo (
                    serverId: "jfrog-server"
                )
            }
        }
        stage('Trivy FS scan') {
                    when {expression {param.action == 'create'}}
            steps{
                sh "trivy fs.> trivyfs.txt"
            }
        }
        stage('Build and Publish Docker Image') {
                    when {expression {param.action == 'create'}}
            steps{
                script {
                    docker.withRegistry ("https://hub.docker.com/", "${params.DockerHubUser}") {
                        docker_image = docker.build "${IMAGE_NAME}"
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push("latest")
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
                    when {expression {param.action == 'create'}}
            steps{
                script {
                    sh """   
                        trivy image ${IMAGE_NAME}:latest > scan.txt
                        cat scan.txt
                    """
                }
            }
        }
        stage('Clean Artifacts') {
                    when {expression {param.action == 'create'}}
            steps{
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }
    }
}