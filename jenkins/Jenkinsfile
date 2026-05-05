pipeline {
    agent any

    environment {
        VERSION = "1.0.${BUILD_NUMBER}"
        ARTIFACT = "hrms-build-${VERSION}.zip"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/tirthmodi2904/hrms-devops-demo.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('sonarserver') {
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=hrms-devops-demo \
                            -Dsonar.sources=. \
                            -Dsonar.login=$SONAR_TOKEN
                            """
                        }
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Artifact') {
            steps {
                sh 'rm -f *.zip'
                sh "zip -r ${ARTIFACT} *.html assets || true"
            }
        }

        stage('Upload Artifact to Nexus (Versioned)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexuslogin',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file ${ARTIFACT} \
                    http://172.31.16.65:8081/repository/hrms-repo/${VERSION}/${ARTIFACT}
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexuslogin',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    -o ${ARTIFACT} \
                    http://172.31.16.65:8081/repository/hrms-repo/${VERSION}/${ARTIFACT}

                    scp ${ARTIFACT} ubuntu@172.31.29.188:/var/www/hrms/

                    ssh ubuntu@172.31.29.188 "
                        cd /var/www/hrms &&
                        rm -rf *.html assets &&
                        unzip -o ${ARTIFACT} &&
                        sudo systemctl restart nginx
                    "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "VERSION ${VERSION} DEPLOYED SUCCESS"
        }
    }
}