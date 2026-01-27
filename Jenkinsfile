pipeline {
    agent { label 'docker-agent' }

    environment {
        SCANNER_HOME = tool 'SonarScanner' 
        DEFECTDOJO_URL = 'http://52.71.8.63:8080' 
        APP_URL = 'http://34.192.61.62'
    }

    stages {
        stage('0. Infrastructure Setup') {
            steps {
                echo "Syncing Infrastructure tools from security-tools folder..."
                checkout scm
                dir('security-tools') {
                    // Ensures Sonar and Postgres are always running and configured
                    sh 'docker-compose -f sonarqube-compose.yml up -d'
                }
            }
        }

        stage('1. Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Ravikiranmasule/cicd_project.git'
            }
        }

        stage('2. Build Backend (JAR)') {
            steps {
                dir('backend-hotellux') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('3. Build Frontend (Angular)') {
            steps {
                dir('frontend-hotellux') {
                    sh 'export NODE_OPTIONS=--openssl-legacy-provider && npm install && npm run build'
                }
            }
        }

        stage('4. SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') { 
                    sh """
                    export SONAR_SCANNER_OPTS="-Xmx512m -Xms256m"
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=HotelLux-Project \
                    -Dsonar.projectName=HotelLux \
                    -Dsonar.sources=. \
                    -Dsonar.java.binaries=backend-hotellux/target/classes
                    """
                }
            }
        }

        stage('5. Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true 
                }
            }
        }

        stage('6. Security Scan (Trivy)') {
            steps {
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/tmp \
                    aquasec/trivy image --severity HIGH,CRITICAL --format json --output /tmp/trivy-report.json mysql:8.0'
            }
        }

        stage('7. Upload Trivy to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'defectdojo-token', variable: 'DOJO_TOKEN')]) {
                    sh """
                    curl -X POST "${DEFECTDOJO_URL}/api/v2/import-scan/" \
                         -H "Authorization: Token $DOJO_TOKEN" \
                         -F "active=true" \
                         -F "verified=false" \
                         -F "scan_type=Trivy Scan" \
                         -F "product_name=HotelLux" \
                         -F "engagement_name=CI/CD Build ${env.BUILD_NUMBER}" \
                         -F "file=@trivy-report.json"
                    """
                }
            }
        }

        stage('8. Docker Deploy') {
            steps {
                sh 'docker-compose down --remove-orphans || true'
                sh 'docker system prune -f'
                sh 'docker-compose up -d --build'
                sh 'sleep 20'
            }
        }

        stage('9. DAST Scan (OWASP ZAP)') {
            steps {
                sh """
                docker run --user root --rm -v \$(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                    -t ${APP_URL} \
                    -J zap-report.json \
                    -r zap-report.html || true
                """
            }
        }

        stage('10. Upload ZAP to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'defectdojo-token', variable: 'DOJO_TOKEN')]) {
                    sh """
                    curl -X POST "${DEFECTDOJO_URL}/api/v2/import-scan/" \
                         -H "Authorization: Token $DOJO_TOKEN" \
                         -H "Content-Type: multipart/form-data" \
                         -F "active=true" \
                         -F "verified=false" \
                         -F "scan_type=ZAP Scan" \
                         -F "product_name=HotelLux" \
                         -F "engagement_name=CI/CD Build ${env.BUILD_NUMBER}" \
                         -F "file=@zap-report.json"
                    """
                }
            }
        }
    }

    post {
        success { echo "SUCCESS: HotelLux DevSecOps Pipeline Finished!" }
        failure { echo "FAILURE: Build failed. Check the Jenkins Console Output." }
    }
}
