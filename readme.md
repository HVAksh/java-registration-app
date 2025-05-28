stage('static code analysis: sonarqube') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                dir('webapp'){
                sh 'mvn -U clean install sonar:sonar'
                sh 'mvn clean package sonar:sonar'
                }
              }
            }
Use package if you're doing quick checks or CI pipelines where full install isn’t necessary.

Use install -U if you're dealing with dependency resolution, want to ensure fresh snapshots, or are working in a multi-module/multi-repo environment.