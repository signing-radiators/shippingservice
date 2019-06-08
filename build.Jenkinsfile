#!groovy

node {
    stage('Clone sources') {
        checkout scm;
    }

    def DOCKER_CONTEXT = "${WORKSPACE}"
    def REPO_NAME = "shippingservice"

    env.BRANCH_NAME = env.BRANCH_NAME ? env.BRANCH_NAME : 'master';
    def imageOwner = "dmitrybuhtiyarov"
    def deployTag = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    def securityScanTag = "securityScan-${env.BUILD_NUMBER}"
    def imageName = "${imageOwner}/${REPO_NAME}"
    def registryFqdn = "registry.hub.docker.com"
    def registryUrl = "https://${registryFqdn}"
    def registryCredentialsId = "dockerhub_id"


    def image;
    docker.withRegistry(registryUrl, registryCredentialsId) {
        stage('Build for security scan') {
            image = docker.build("${imageName}:${securityScanTag}", DOCKER_CONTEXT);
        }
        image.inside("--entrypoint=''") {
            stage('Sonarqube') {
                withSonarQubeEnv('sonarqube') {
                    def scannerHome = tool name: 'sonarqube'
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }

            stage("Quality Gate") {
                timeout(time: 5, unit: 'MINUTES') {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                }
            }
        }

        stage('Push for security scan') {
            image.push();
        }
    }

    stage('Security scan') {
        sh "echo  '${registryFqdn}/${imageName}:${securityScanTag} ${DOCKER_CONTEXT}/Dockerfile' > anchore_images"
        anchore bailOnFail: false, forceAnalyze: true, name: 'anchore_images'
    }

    docker.withRegistry(registryUrl, registryCredentialsId) {
        stage('Push') {
            image.push("${deployTag}");
        }
    }

}
