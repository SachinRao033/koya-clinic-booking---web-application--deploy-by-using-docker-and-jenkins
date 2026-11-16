
pipeline {
    agent any

    environment {
        APP_NAME = "koya-clinic-app"
        APP_PORT = "8000"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Test Dependencies') {
            steps {
                sh '''
                    python3 -m venv .venv
                    .venv/bin/pip install --upgrade pip
                    .venv/bin/pip install -r requirements.txt
                    .venv/bin/pip install pytest httpx
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    .venv/bin/pytest -v
                '''
            }
        }

        stage('Prepare Environment') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'koya-api-key',
                        variable: 'API_KEY'
                    )
                ]) {
                    sh '''
                        set +x
                        umask 077

                        cat > .env <<EOF
DATABASE_URL=sqlite:////data/clinic.db
API_KEY=${API_KEY}
SEED_DEMO_SLOTS=true
EOF
                    '''
                }
            }
        }

        stage('Docker Compose Validation') {
            steps {
                sh '''
                    docker compose config --quiet
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'koya-api-key',
                        variable: 'API_KEY'
                    )
                ]) {
                    sh '''
                        docker compose build --no-cache
                    '''
                }
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    docker compose up -d
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'koya-api-key',
                        variable: 'API_KEY'
                    )
                ]) {
                    sh '''
                        echo "Checking application..."

                        for i in $(seq 1 30); do
                            if curl -fsS http://localhost:8000/docs >/dev/null; then
                                echo "Application is responding!"
                                break
                            fi

                            echo "Waiting for application..."
                            sleep 2
                        done

                        curl -fsS http://localhost:8000/docs >/dev/null

                        echo "Checking API..."

                        curl -fsS \
                            -H "X-API-Key: ${API_KEY}" \
                            http://localhost:8000/slots \
                            -o /dev/null

                        echo "API verification successful!"
                    '''
                }
            }
        }

    }

    post {
        success {
            echo 'Koya Clinic Booking deployed successfully!'

            sh '''
                docker compose ps
            '''
        }

        failure {
            echo 'Deployment failed. Checking container logs...'

            sh '''
                docker compose logs --tail=100 || true
            '''
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
