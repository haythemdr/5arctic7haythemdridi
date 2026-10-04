pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Test Backend') {
            steps {

                // Start a temporary MySQL container for the tests
                sh '''
                    docker rm -f jenkins-test-mysql 2>/dev/null || true

                    docker run -d \
                      --name jenkins-test-mysql \
                      -e MYSQL_ROOT_PASSWORD=root \
                      -e MYSQL_DATABASE=test_db \
                      -p 3307:3306 \
                      mysql:8.0

                    echo "Waiting for MySQL to become ready..."

                    until docker exec jenkins-test-mysql \
                      mysqladmin ping -h localhost -proot --silent
                    do
                        echo "MySQL is not ready yet..."
                        sleep 5
                    done

                    echo "MySQL is ready!"
                '''

                // Run Spring Boot tests using the temporary MySQL
                dir('backend') {
                    sh '''
                        SPRING_DATASOURCE_URL="jdbc:mysql://localhost:3307/test_db?createDatabaseIfNotExist=true" \
                        SPRING_DATASOURCE_USERNAME=root \
                        SPRING_DATASOURCE_PASSWORD=root \
                        mvn test
                    '''
                }
            }

            post {
                always {
                    sh '''
                        echo "Removing temporary MySQL test container..."
                        docker rm -f jenkins-test-mysql 2>/dev/null || true
                    '''
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh '''
                        docker run --rm \
                          -v "$PWD:/app" \
                          -w /app \
                          node:22-alpine \
                          sh -c "npm ci && npm run build"
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    echo "Building Docker images..."
                    docker compose build
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "Stopping previous application stack..."
                    docker compose down || true

                    echo "Starting application stack..."
                    docker compose up -d
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    echo "Waiting for Spring Boot to start..."

                    sleep 120

                    echo "Docker services:"
                    docker compose ps

                    echo "Testing backend..."
                    curl -f http://localhost:8081/entreprise/all

                    echo "Testing frontend -> backend proxy..."
                    curl -f http://localhost:4200/api/entreprise/all

                    echo "Application verification successful!"
                '''
            }
        }
    }

    post {

        success {
            echo 'CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD pipeline failed. Check the stage logs.'
        }

        always {
            sh '''
                docker rm -f jenkins-test-mysql 2>/dev/null || true
            '''
        }
    }
}
