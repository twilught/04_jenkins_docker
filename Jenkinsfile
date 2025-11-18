pipeline {
    agent any

    options {
        durabilityHint 'MAX_SURVIVABILITY'   // ⭐ ป้องกัน Jenkins restart kill pipeline
        timeout(time: 60, unit: 'MINUTES')   // กัน build ค้างตาย
        timestamps()                         // Log อ่านง่ายขึ้น
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        BUILD_TAG = "${env.BUILD_NUMBER}"
    }

    parameters {
        booleanParam(
            name: 'CLEAN_VOLUMES',
            defaultValue: true,
            description: 'Remove volumes (clears database)'
        )
        string(
            name: 'API_HOST',
            defaultValue: 'http://72.60.236.166:3001',
            description: 'Frontend will call this API URL'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    checkout scm
                    env.GIT_COMMIT_SHORT = sh(returnStdout: true,
                        script: 'sleep 1 && git rev-parse --short HEAD').trim()
                    echo "Commit: ${env.GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('Prepare Environment') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'MYSQL_ROOT_PASSWORD', variable: 'MYSQL_ROOT_PASS'),
                        string(credentialsId: 'MYSQL_PASSWORD', variable: 'MYSQL_PASS')
                    ]) {

                        writeFile file: '.env', text: """\
MYSQL_ROOT_PASSWORD=${env.MYSQL_ROOT_PASS}
MYSQL_DATABASE=attractions_db
MYSQL_USER=attractions_user
MYSQL_PASSWORD=${env.MYSQL_PASS}

DB_HOST=mysql
DB_PORT=3306
API_PORT=3001
FRONTEND_PORT=3000

NODE_ENV=production
API_HOST=${params.API_HOST}
TZ=Asia/Bangkok
""".stripIndent()

                        echo ".env created"
                    }
                }
            }
        }

        stage('Validate Compose') {
            steps {
                script {
                    echo "Validating Docker Compose..."
                    sh 'sleep 1 && docker compose config'
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def downCmd = params.CLEAN_VOLUMES ?
                        "docker compose down -v" :
                        "docker compose down"

                    // ⭐ sleep 1 ป้องกัน wrapper script error
                    sh "sleep 1 && ${downCmd}"

                    sh """
                        sleep 1
                        docker compose build --no-cache
                        sleep 1
                        docker compose up -d
                    """

                    echo "Deployment completed"
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "Waiting for API…"
                    sh 'sleep 15'

                    sh """
                        sleep 1
                        timeout 60 bash -c 'until curl -f http://localhost:3001/health; do sleep 2; done'
                        curl -f http://localhost:3001/attractions
                        echo "Health check OK!"
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    sh """
                        sleep 1
                        docker compose ps
                        echo "=== Logs ==="
                        docker compose logs --tail=20

                        echo "Frontend:  http://localhost:3000"
                        echo "API:        http://localhost:3001"
                        echo "phpMyAdmin: http://localhost:8888"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed — showing logs"
            sh 'sleep 1 && docker compose logs --tail=50 || true'
        }
        always {
            sh """
                sleep 1
                docker image prune -f
                docker container prune -f
            """
        }
    }
}
