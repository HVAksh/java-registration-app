pipeline{
    agent any
    tools{
        maven 'maven'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = 'jave-registration-app'
        RELEASE = '1.0.0'
        DOCKER_USER = 'hvaksh'
        IMAGE_NAME = '${DOCKER_USER}'+'/'+'${APP_NAME}'
        IMAGE_TAG = '${RELEASE}-${BUILD_NUMBER}'
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
                    id: 'jfrog-server'
                    url: http://192.168.147.128:8082/artifactory
                    credentialsId: 'jfrog'
                )
                rtMavenDeployer (
                    id: 'MAVEN_DEPLOYER',
                    serverId: 'jfrog-server',
                    releaseRepo: 'libs-release-local',
                    snapshotRepo: 'libs-snapshot-local'
                )
                rtMavenResolver (
                    id: 'MAVEN_RESOLVER',
                    serverId: 'jfrog-server',
                    releaseRepo: 'libs-release',
                    snapshotRepo: 'libs-snapshot'
                )
            }
        }
        stage('Deploy artifacts') {
            steps {
                rtMavenRun (
                    tool: 'maven', 
                    pom: 'webapp/pom.xml',
                    goals: 'clean install',
                    deployerId: 'MAVEN_DEPLOYER',
                    resolverId: 'MAVEN_RESOLVER'
                )
            }
        }
        stage('build publish info') {
            steps {
                rtPublishBuildInfo (
                    serverId: 'jfrog-server'
                )
            }
        }
        tage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
         }
         stage("Build & Push Docker Image") {
             steps {
                 script {
                     docker.withRegistry('',DOCKER_PASS) {
                         docker_image = docker.build "${IMAGE_NAME}"
                     }
                     docker.withRegistry('',DOCKER_PASS) {
                         docker_image.push("${IMAGE_TAG}")
                         docker_image.push('latest')
                     }
                 }
             }
         }
         stage("Trivy Image Scan") {
             steps {
                 script {
	                  sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ashfaque9x/java-registration-app:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table > trivyimage.txt')
                 }
             }
         }
         stage ('Cleanup Artifacts') {
             steps {
                 script {
                      sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                      sh "docker rmi ${IMAGE_NAME}:latest"
                 }
             }
         }
         stage('Deploy to Kubernets'){
             steps{
                 script{
                      dir('Kubernetes') {
                         kubeconfig(credentialsId: 'kubernetes', serverUrl: '') {
                         sh 'kubectl apply -f deployment.yml'
                         sh 'kubectl apply -f service.yml'
                         sh 'kubectl rollout restart deployment.apps/registerapp-deployment'
                         }   
                      }
                 }
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
            to: 'ashfaque.s510@gmail.com',                              
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
      }
    }
}