@Library('shared_lib') _
def registry = 'https://pramoth28.jfrog.io/'
pipeline {
    agent any

    parameters {
        choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
        string(name: 'ImageName', description: "name of the docker build", defaultValue: 'javapp')
        string(name: 'ImageTag', description: "tag of the docker build", defaultValue: 'v1')
        string(name: 'DockerHubUser', description: "name of the Application", defaultValue: 'promo286')
    }

    stages {
        stage('Git Checkout') {
            when { expression {  params.action == 'create' } }
            steps {
                gitCheckout(
                    branch: "main",
                    url: "https://github.com/praveen1994dec/Java_app_3.0.git"
                )
            }
        }

        stage('Unit Test maven') {
            when { expression {  params.action == 'create' } }
            steps {
                script {
                    mvnTest()
                }
            }
        }

        stage('Integration Test maven') {
            when { expression {  params.action == 'create' } }
            steps {
                script {
                    mvnIntegrationTest()
                }
            }
        }

        stage('Static code analysis: Sonarqube') {
            environment {
                scannerHome = tool 'sonar-tool'
            }
            when { expression {  params.action == 'create' } }
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=promoth-28_jenkins-ci \
                            -Dsonar.organization=promoth-28 \
                            -Dsonar.projectName=Jenkins-ci \
                            -Dsonar.language=java \
                            -Dsonar.sourceEncoding=UTF-8 \
                            -Dsonar.sources=. \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml"
                    }
                }
            }
        }

        stage('Quality Gate Status Check : Sonarqube') {
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

        stage('Maven Build : maven') {
            when { expression {  params.action == 'create' } }
            steps {
                script {
                    mvnBuild()
                }
            }
        }
        stage('jar Publish ') {
            steps {
                script {
                    echo '<--------------- Jar Publish Started --------------->'
                    def server = Artifactory.newServer url: registry + "/artifactory", credentialsId: "jfrog-cred"
                    def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}";
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "target/(*)",
                                "target": "libs-release-local/{1}",
                                "flat": "false",
                                "props": "${properties}",
                                "exclusions": ["*.sha1", "*.md5"]
                            }
                        ]
                    }"""
                    def buildInfo = server.upload(uploadSpec)
                    buildInfo.env.collect()
                    server.publishBuildInfo(buildInfo)
                    echo '<--------------- Jar Publish Ended --------------->'
                }
            }
        }
        stage('Docker Image Build') {
            when { expression {  params.action == 'create' } }
            steps {
                script {
                    dockerBuild("${params.ImageName}","${params.ImageTag}","${params.DockerHubUser}")
                }
            }
        }

        stage('Docker Image Scan: trivy ') {
            when { expression {  params.action == 'create' } }
            steps {
                script {
                    dockerImageScan("${params.ImageName}","${params.ImageTag}","${params.DockerHubUser}")
                }
            }
        }

        stage('Docker Image Push : DockerHub ') {
            when { expression {  params.action == 'create' } }
            steps {
                script {
                    dockerImagePush("${params.ImageName}","${params.ImageTag}","${params.DockerHubUser}")
                }
            }
        }

        stage('Docker Image Cleanup : DockerHub ') {
            when { expression {  params.action == 'create' } }
            steps {
                script {
                    dockerImageCleanup("${params.ImageName}","${params.ImageTag}","${params.DockerHubUser}")
                }
            }
        }
    }
}
