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
                    sh './mvnw clean verify'
                }
            }
        }

        stage('Cart and Catalog Sequential Build') {
            stages {
               stage('Cart Build and Test') {
                  steps {
                     dir('application/src/cart') {
                         withEnv([
                            'AWS_ACCESS_KEY_ID=dummy',
                            'AWS_SECRET_ACCESS_KEY=dummy',
                            'AWS_REGION=us-west-2'
                        ]) {
                            sh './mvnw clean verify'
                        }
                    }
               }
        }

                stage('Catalog Build and Test') {
                    steps {
                        dir('application/src/catalog') {
                            sh '''#!/bin/bash
                               mkdir -p test-results
                               set -o pipefail
                              go test -count=1 -coverpkg=./... -coverprofile=test-results/catalog-coverage.out -v ./... 2>&1 | tee test-results/catalog-test.log
                              "$HOME/go/bin/go-junit-report" -set-exit-code < test-results/catalog-test.log > test-results/catalog-junit.xml
                            '''
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

                    sh 'mkdir -p test-results && rm -f test-results/junit.xml'
                    sh 'JEST_JUNIT_OUTPUT_FILE=test-results/junit.xml yarn test:integration --reporters=default --reporters=jest-junit'

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

        stage('SonarQube Analysis - UI') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube-Local') {
                        dir('application/src/ui') {
                            sh """
                                "${scannerHome}/bin/sonar-scanner" \
                                    -Dsonar.projectKey=retail-ui \
                                    -Dsonar.projectName=retail-ui \
                                    -Dsonar.sources=src/main \
                                    -Dsonar.tests=src/test \
                                    -Dsonar.java.binaries=target/classes \
                                    -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                            """
                        }
                    }
                }
            }
       }


       stage('SonarQube Quality Gate - UI') {
           steps {
              timeout(time: 5, unit: 'MINUTES') {
                 waitForQualityGate abortPipeline: true
              }
           }
       }


       stage('SonarQube Analysis - Cart') {
           steps {
               script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube-Local') {
                        dir('application/src/cart') {
                             sh """
                                 "${scannerHome}/bin/sonar-scanner" \
                                    -Dsonar.projectKey=retail-cart \
                                    -Dsonar.projectName=retail-cart \
                                    -Dsonar.sources=src/main \
                                    -Dsonar.tests=src/test \
                                    -Dsonar.java.binaries=target/classes
                             """
                        }
                    }
                }
            }
       }


       stage('SonarQube Quality Gate - CART') {
           steps {
               timeout(time: 5, unit: 'MINUTES') {
                  waitForQualityGate abortPipeline: true
               }
           }
       }



      stage('SonarQube Analysis - Catalog') {
         steps {
             script {
                 def scannerHome = tool 'SonarScanner'

                 withSonarQubeEnv('SonarQube-Local') {
                     dir('application/src/catalog') {
                         sh """
                             "${scannerHome}/bin/sonar-scanner" \
                               -Dsonar.projectKey=retail-catalog \
                               -Dsonar.projectName=retail-catalog \
                               -Dsonar.sources=. \
                               -Dsonar.tests=test \
                               -Dsonar.exclusions=test/**,test-results/**,vendor/** \
                               -Dsonar.test.inclusions=**/*_test.go \
                               -Dsonar.go.coverage.reportPaths=test-results/catalog-coverage.out
                          """
                      }
                 }
             }
         }
      }

       stage('SonarQube Quality Gate - Catalog') {
           steps {
               timeout(time: 5, unit: 'MINUTES') {
                  waitForQualityGate abortPipeline: true
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

                                ./mvnw -B -s "$MAVEN_SETTINGS" deploy -DskipTests \
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
              junit testResults: 'application/src/**/target/surefire-reports/TEST-*.xml,application/src/catalog/test-results/*.xml,application/src/checkout/test-results/*.xml',
                    allowEmptyResults: true
          }
      }
}
