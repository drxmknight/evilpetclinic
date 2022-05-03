pipeline {
    agent any
    // This should be in the jenkins envs !!!
    environment {
        BITBUCKET_REPO = 'http://10.3.6.21:7990/scm/eh/evilpetclinic.git'
        USER_NAME =  'ehales'
        REPO_NAME = 'evilpetclinic'
        IMAGE_NAME = "${USER_NAME}/${REPO_NAME}:latest"
    }
    stages {
        // Checkout git repository from local bitbucket
        stage('Git Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: 'main']], userRemoteConfigs: [[credentialsId: 'bitbucket-credential', url: BITBUCKET_REPO]]])
            }
        }
        // scan local repo with prisma cloud jenkins plugin (twistcli coderepo scan)
        stage('Prisma coderepo scan') {
            steps {
                prismaCloudScanCode excludedPaths: '', explicitFiles: '', logLevel: 'debug', pythonVersion: '', 
                repositoryName: REPO_NAME, repositoryPath: '.', resultsFile: 'prisma-cloud-repo-scan-results.json'
            }
        }
        // Build the image from the repo Dockerfile
        stage('Docker build evilpetclinic') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }
        // Scan image with prisma cloud jenkins plugin (twistcli image scan)
        stage('Prisma scan image') {
            steps {
                prismaCloudScanImage ca: '', cert: '', dockerAddress: 'unix:///var/run/docker.sock', ignoreImageBuildTime: true, image: IMAGE_NAME,
                key: '', logLevel: 'debug', podmanPath: '', project: '', resultsFile: 'prisma-cloud-image-scan-results.json'
            }
        }
        // Build the bridgecrew image to use as agent to scan IaC
        stage('Build Bridgecrew image') {
            steps {
                sh 'echo "FROM alpine:latest" > Dockerfile'
                sh 'echo "RUN apk add --no-cache python3 py3-pip gcc musl-dev python3-dev libffi-dev" >> Dockerfile'
                sh 'echo "RUN pip3 install --upgrade pip && pip3 install setuptools wheel" >> Dockerfile'
                sh 'echo "RUN pip3 install bridgecrew" >> Dockerfile'
                sh 'docker build -t bridgecrew:latest .'
            }
        }
        // Checkout the repo and scan it with bridgecrew for IaC misconfigurations
        stage('Scan Kubernetes manifest with Bridgecrew') {
            agent {
                docker {
                    image 'bridgecrew:latest'
                }
            }
            environment {
                PRISMA_API_URL = "https://api2.prismacloud.io"
                PRISMA_TOKEN = credentials('prisma-token')
            }
            steps {
                checkout([$class: 'GitSCM', branches: [[name: 'main']], userRemoteConfigs: [[credentialsId: 'bitbucket-credential', url: BITBUCKET_REPO]]])
                // Export the results to junitxml for jenkins. 
                // Important: this scan check policies configured in the prisma console, and will fail if any policy does not pass. 
                // It is not posible to configure rules for this. If need to pass, add " || true" at the final of the script.
                sh "bridgecrew --directory . --bc-api-key $PRISMA_TOKEN_USR::$PRISMA_TOKEN_PSW --repo-id ${IMAGE_NAME} -o junitxml > result.xml || true"
                junit 'result.xml'
            }
        }
        // Symbolic deployment
        stage('Deploy evilpetclinic') {
            steps {
                sh 'echo "deploying with kubectl..."'
            }
        }
    }
    post {
        // Always publish the results to prisma cloud console.
        always {
            prismaCloudPublish resultsFilePattern: 'prisma-cloud-image-scan-results.json'
            prismaCloudPublish resultsFilePattern: 'prisma-cloud-repo-scan-results.json'
        }
    }
    options {
        preserveStashes()
        timestamps()
    }
}

