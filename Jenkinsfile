
pipeline {
    agent { label 'linux' }

    stages {
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
    }

    post {
        always {
            junit 'application/src/ui/target/surefire-reports/TEST-*.xml'
        }
    }
}
