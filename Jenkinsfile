def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

pipeline {
    agent any;

    tools {
        maven "maven"
        jdk "OracleJDK17"
    }

    environment {
        DOCKERHUB_CREDS = credentials('dockerhub')
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
        APP_REPO = 'rajatrulaniya/app'
    }

    stages {
        stage("Build") {
          steps {
            sh "mvn clean package -DskipTests"
          }
          post {
              success {
                  echo 'Now Archiving...'
                  archiveArtifacts artifacts: '**/target/*.war'
              }
          }
        }

        stage("Test") {
            steps {
                sh "mvn clean test -Djacoco.skip=true"
            }
        }

        stage("Checkstyle Analysis") {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }    

        stage("Sonar Analysis") {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
               withSonarQubeEnv("${SONARSERVER}") {
                   sh '''
                        ${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vproapp-cicd \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                   '''
              }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: "HOURS") {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
          steps {
            sh 'docker build -t $APP_REPO:v$BUILD_ID -t $APP_REPO:latest .'
          }
        }

        stage('Login to DockerHub') {
          steps {
            sh 'echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin'
          }
        }

        stage('Push Image') {
          steps {
            sh 'docker push --all-tags ${APP_REPO}'

            sh 'docker rmi -f $(docker images ${APP_REPO} -q)'
          }
        }

        stage('Kubernetes Deploy') {
          agent {label 'KOPS'} 

          steps {
            checkout scm

            sh 'sudo helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appImage=${APP_REPO}:latest --namespace prod'
          }
        }
    }

    post {
        success {
            echo 'Slack Notification....'
            slackSend channel: '#jenkinscicd', 
                color: COLOR_MAP[currentBuild.currentResult], 
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build-Version: Build-${env.BUILD_ID}_${env.BUILD_TIMESTAMP} \n\n More info at: ${env.BUILD_URL}"
        }
    }
}