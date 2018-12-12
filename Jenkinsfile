pipeline {
    agent any
    options {
      buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    stages {
        stage('Init') {
            steps {
                echo "Building $BRANCH_NAME on $JENKINS_URL ..."
                withCredentials([file(credentialsId: 'innov-atm-verdaccio', variable: 'NPMRC')]) {
                    sh '''
                    docker run -t -v $PWD:/app -v $NPMRC:/app/.npmrc -w /app node:10 npm install -g yarn
                    docker run -t -v $PWD:/app -v $NPMRC:/app/.npmrc -w /app node:10 yarn install
                    '''
                }
            }
        }
        stage('Release version check') {
            when {
                tag "v*.*.*"
            }
            steps {
                script {
                    def packageVersion = sh script: 'cat package.json | grep version | head -1 | awk -F: \'{ print $2 }\' | sed \'s/[",]//g\' ' , returnStdout: true
                    packageVersion = 'v' + packageVersion.replaceAll("\\s","")

                    def tagVersion = sh script: 'git tag --sort version:refname | tail -1' , returnStdout: true
                    tagVersion = tagVersion.replaceAll("\\s","")
                    if (packageVersion != tagVersion) {
                        echo "[FAILURE] Build stopped due to an incoherent version between package.json (${packageVersion}) and tag (${tagVersion})"
                        sh "exit 1"
                    }
                }
            }
        }
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('InnovATM SonarQube') {
                    script {
                        def scannerHome = tool 'SonarQube Scanner 3.2'
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
        stage("SonarQube QG") {
            steps {
                script {
                    def qg = waitForQualityGate()
                    SONAR_STATUS=qg.status
                }
            }
        }
        stage('Build') {
            steps {
                withCredentials([file(credentialsId: 'innov-atm-verdaccio', variable: 'NPMRC')]) {
                  sh '''
                  docker run -t -v $PWD:/app -v $NPMRC:/app/.npmrc -w /app node:10 yarn build
                  '''
              }
            }
        }
        stage('Publish') {
            when {
                tag "v*.*.*"
            }
            steps {
                withCredentials([file(credentialsId: 'innov-atm-verdaccio', variable: 'NPMRC')]) {
                  sh '''
                  docker run -t -v $PWD:/app -v $NPMRC:/app/.npmrc -w /app node:10 npm publish --force --registry https://verdaccio.innov-atm.com
                  '''
              }
            }
        }
    }
}