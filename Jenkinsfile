pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node23'
    }
    environment{
        SCANNER_HOME = "${tool 'sonar-scanner'}"
    }
    stages{
        stage('Clean Workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Git Checkout'){
            steps{
                git 'https://github.com/sdeep1096/DevOps-Project-Zomato-Kastro.git'
            }
        }
        stage('Sonarqube Analysis'){
            steps{
                withSonarQubeEnv('sonar-server'){
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage('Code Quality Gate'){
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }
        stage('Install NPM dependencies'){
            steps{
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                script {
                    sh 'dependency-check.sh --updateOnly || true'
                }
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit -n', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        }
        stage('Trivy File Scan'){
            steps{
                sh "trivy fs . | tee trivy.txt"
            }
        }
        stage('Build Docker Image'){
            steps{
                sh "docker build -t briarheart1096/zomato:latest ."
            }
        }
        stage("Push Image to Dockerhub"){
            steps{
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push briarheart1096/zomato:latest"
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker'){
                       sh 'docker-scout quickview briarheart1096/zomato:latest'
                       sh 'docker-scout cves briarheart1096/zomato:latest'
                       sh 'docker-scout recommendations briarheart1096/zomato:latest'
                   }
                }
            }
        }
        stage('Deploy Container'){
            steps{
                sh "docker run -d -p 3000:3000 --name zomato briarheart1096/zomato:latest"
            }
        }
    }
post {
    unstable {
            script {
                echo "OWASP Dependency-Check found vulnerabilities. Marking build as unstable."
            }
        }
        failure {
            script {
                echo "OWASP Dependency-Check failed due to missing database or another critical issue."
            }
        }
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: """
                <html>
                <body>
                    <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                    </div>
                    <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                    </div>
                    <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                    </div>
                </body>
                </html>
            """,
            to: 'sdipghosh1096@gmail.com',
            mimeType: 'text/html',
            attachmentsPattern: 'trivy.txt'
    }
}
}
