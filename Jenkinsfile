pipeline {
    agent {
        node {
            label 'jenkins-slave-node'
        }
    }
    environment {
        PATH = "/opt/apache-maven-3.9.6/bin:$PATH"
    }
    stages {
        stage("build code"){
            steps {
                echo "----------- build started ----------"
                sh 'mvn clean deploy -Dmaven.test.skip=true'
                echo "----------- build completed ----------"
            }
        }
        stage("test stage"){
            steps{
                echo "----------- unit test started ----------"
                sh 'mvn surefire-report:report'
                echo "----------- unit test Completed ----------"
            }
        }

        stage('SonarQube analysis') {
            environment {
                scannerHome = tool 'sonar-scanner-meportal'
            }
            steps{
                withSonarQubeEnv('sonar-server-meportal') {
                sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }
        stage("Quality Gate"){
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
        stage("Artifact Publish") {
            steps {
                script {
                    echo '------------- Artifact Publish Started ------------'
                    def server = Artifactory.newServer url:"https://debiproject.jfrog.io//artifactory" ,  credentialsId:"jrog-creden"
                    def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}";
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "staging/(*)",
                                "target": "release-local-artifacts-2/{1}",
                                "flat": "false",
                                "props" : "${properties}",
                                "exclusions": [ "*.sha1", "*.md5"]
                            }
                        ]
                    }"""
                    def buildInfo = server.upload(uploadSpec)
                    buildInfo.env.collect()
                    server.publishBuildInfo(buildInfo)
                    echo '------------ Artifact Publish Ended ------------'
                }
            }
        }
        stage(" Create Docker Image ") {
            steps {
                script {
                    echo '-------------- Docker Build Started -------------'
                    app = docker.build("debiproject.jfrog.io/docker-2-docker-local/myapp:1.0.1")
                    echo '-------------- Docker Build Ended -------------'
                }
            }
        }
        stage (" Docker Publish "){
            steps {
                script {
                        echo '---------- Docker Publish Started --------'
                        docker.withRegistry("https://debiproject.jfrog.io", 'jrog-creden'){
                        app.push()
                        echo '------------ Docker Publish Ended ---------'
                    }
                }
            }
        }
        stage ("Deploy Stage"){
            steps {
                script {
                    sh './deploy.sh'
                }
            }
        }
    }

}

