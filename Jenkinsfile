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
                git branch: 'main',
                    credentialsId: 'github-retail-store-ci',
                    url: 'git@github.com:PaponMiah/aws-retail-devops-platform.git'
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
                stage('Cart Compile') {
                    steps {
                        dir('application/src/cart') {
                            sh './mvnw clean compile'
                        }
                    }
                }

                stage('Catalog Build') {
                    steps {
                        dir('application/src/catalog') {
                            sh 'go build -o main main.go'
                        }
                    }
                }
            }
        }

        stage('Checkout Build') {
            steps {
                dir('application/src/checkout') {
                    sh 'yarn install'
                    sh 'yarn build'
                }
            }
        }

        stage('Orders Compile') {
            steps {
                dir('application/src/orders') {
                    sh './mvnw clean compile'
                }
            }
        }

        stage('Publish Cart to Nexus') {
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



    post {
        always {
            junit 'application/src/ui/target/surefire-reports/TEST-*.xml'
        }
    }
}
