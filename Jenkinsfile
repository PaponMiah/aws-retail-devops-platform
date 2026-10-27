pipeline {
    agent { label 'linux' }

    environment {
         APP_NAME = 'retail-store'
         BUILD_ENV = 'development'
    }

    parameters {
        choice(
           name: 'DEPLOY_ENV',
           choices: ['development', 'staging', 'production'],
           description: 'Select the target environment'
        )
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {

        stage('Verify Environment') {
            steps {
                sh 'echo "Application Name: $APP_NAME"'
                sh 'echo "Build Environment: $BUILD_ENV"'
                echo "Selected Environment: ${params.DEPLOY_ENV}"
                //sh 'echo "Selected Environment: $DEPLOY_ENV"'
            }
        }

        stage('Parallel Verification') {
            parallel {
                stage('Check Java') {
                    steps {
                        sh 'java -version'
                    }
                }

                stage('Check Git') {
                    steps {
                        sh 'git --version'
                    }
                }
            }
        }

        stage('Staging Verification') {
            when {
                expression { params.DEPLOY_ENV == 'staging' }
            }

            steps {
                echo 'Staging env selected'
            }
        }

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Verify Source') {
            steps {
                sh 'git log -1 --oneline'
                sh 'ls application/src'
            }
        }

        stage('UI Build and Test') {
            steps {
                dir('application/src/ui') {
                    sh './mvnw clean test'
                }
            }
        }

        stage('Cart and Catalog Parallel Build') {
            parallel {
               stage('Cart Build and Test') {
                  steps {
                     dir('application/src/cart') {
                         withEnv([
                            'AWS_ACCESS_KEY_ID=dummy',
                            'AWS_SECRET_ACCESS_KEY=dummy',
                            'AWS_REGION=us-west-2'
                        ]) {
                            sh './mvnw clean package'
                        }
                    }
               }
        }

                stage('Catalog Build and Test') {
                    steps {
                        dir('application/src/catalog') {

                            sh 'go test -v ./...'
                            sh 'go build -o main main.go'
                        }
                    }
                }
            }
        }

        stage('Checkout Build and Test') {
            steps {
                dir('application/src/checkout') {
                    sh 'yarn install --immutable'
                    sh 'yarn test:integration'
                    sh 'yarn build'
                }
            }
        }

        stage('Orders Build and Test') {
            steps {
                dir('application/src/orders') {
                    sh './mvnw clean package'
                }
            }
        }

        stage('Publish Cart to Nexus') {
           
             when {
                expression {
                    env.BRANCH_NAME == null || env.BRANCH_NAME == 'main'
                }
             }

               steps {
                dir('application/src/cart') {

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'nexus-maven-publisher',
                            usernameVariable: 'NEXUS_USER',
                            passwordVariable: 'NEXUS_PASS'
                        )
                    ]) {

                        configFileProvider([
                            configFile(
                                fileId: 'ede0529a-cfcb-4bb4-ac59-c24ee53fd928',
                                variable: 'MAVEN_SETTINGS'
                            )
                        ]) {

                            sh '''
                                set +x

                                ./mvnw -B -s "$MAVEN_SETTINGS" clean deploy -DskipTests \
                                  -DaltSnapshotDeploymentRepository=nexus-snapshots::http://192.168.178.28:8081/repository/maven-snapshots/
                            '''
                        }
                    }
                }
            }
        }
  


       stage('Archive Cart Artifact') {
           steps {
              archiveArtifacts(
                 artifacts: 'application/src/cart/target/*.jar',
                 fingerprint: true,
                 allowEmptyArchive: false
              )
          }
      }

}

       post {
          always {
              junit testResults: 'application/src/{ui,cart,orders}/target/surefire-reports/TEST-*.xml',
                    allowEmptyResults: true
          }
      }
}
