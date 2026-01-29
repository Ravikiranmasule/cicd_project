pipeline {
    agent { label 'docker-agent' }

    options {
        skipDefaultCheckout()
        timeout(time: 1, unit: 'HOURS')
    }

    environment {
        SCANNER_HOME = tool 'SonarScanner' 
        DEFECTDOJO_URL = 'http://52.71.8.63:8080' 
        APP_URL = 'http://angular-frontend'
    }

    stages {
        stage('0. Permission & Memory Cleanup') {
            steps {
                echo "Cleaning memory safely without wiping tool data..."
                sh 'sudo systemctl enable docker'
                
                // Forces running containers to restart, but doesn't delete them
                sh 'docker update --restart always $(docker ps -q) || true'
                
                // SAFE CLEAN: Removed '-a'. This keeps your tool images (Sonar/Dojo/MySQL) safe.
                sh 'docker image prune -f' 
                
                sh 'sudo chmod -R 777 ${WORKSPACE} || true'
                checkout scm
                
                dir('security-tools') {
                    // Starts tools only if they aren't already running
                    sh 'docker-compose -f sonarqube-compose.yml up -d'
                }
            }
        }

        stage('1. Build Backend (JAR)') {
            steps {
                dir('backend-hotellux') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('2. Build Frontend (Angular)') {
            steps {
                dir('frontend-hotellux') {
                    sh 'export NODE_OPTIONS=--openssl-legacy-provider && npm install && npm run build'
                }
            }
        }

        stage('3. SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') { 
                    sh """
                    export SONAR_SCANNER_OPTS="-Xmx512m"
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=HotelLux-Project \
                    -Dsonar.projectName=HotelLux \
                    -Dsonar.sources=. \
                    -Dsonar.java.binaries=backend-hotellux/target/classes \
                    -Dsonar.javascript.node.maxspace=1024
                    """
                }
            }
        }

        stage('4. Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true 
                }
            }
        }

        stage('5. Security Scan (Trivy)') {
            steps {
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/tmp \
                    aquasec/trivy image --severity HIGH,CRITICAL --format json --output /tmp/trivy-report.json mysql:8.0'
            }
        }

        stage('6. Create Engagement & Upload Trivy') {
            steps {
                withCredentials([string(credentialsId: 'defectdojo-token', variable: 'DOJO_TOKEN')]) {
                    sh """
                    curl -X POST "${DEFECTDOJO_URL}/api/v2/engagements/" \
                         -H "Authorization: Token \$DOJO_TOKEN" \
                         -H "Content-Type: multipart/form-data" \
                         -F "name=CI/CD Build ${env.BUILD_NUMBER}" \
                         -F "target_start=\$(date +%Y-%m-%d)" \
                         -F "target_end=\$(date -d '+1 day' +%Y-%m-%d)" \
                         -F "product=1" \
                         -F "status=In Progress" \
                         -F "engagement_type=CI/CD"

                    curl -X POST "${DEFECTDOJO_URL}/api/v2/import-scan/" \
                         -H "Authorization: Token \$DOJO_TOKEN" \
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

        stage('7. Docker Deploy') {
            steps {
                // ADDED: Force removes any existing prometheus container to avoid naming conflicts
                sh 'docker rm -f prometheus || true' 
                sh 'docker-compose down --remove-orphans || true'
                sh 'docker-compose up -d --build'
                sh 'sleep 30' 
            }
        }

        stage('8. DAST Scan (OWASP ZAP)') {
            steps {
                sh """
                docker run --user root --network hotellux-app-build_hotel-network --rm -v \$(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                    -t ${APP_URL} \
                    -x zap-report.xml \
                    -r zap-report.html || true
                """
            }
        }

        stage('9. Upload ZAP to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'defectdojo-token', variable: 'DOJO_TOKEN')]) {
                    sh """
                    curl -X POST "${DEFECTDOJO_URL}/api/v2/import-scan/" \
                         -H "Authorization: Token \$DOJO_TOKEN" \
                         -H "Content-Type: multipart/form-data" \
                         -F "active=true" \
                         -F "verified=false" \
                         -F "scan_type=ZAP Scan" \
                         -F "product_name=HotelLux" \
                         -F "engagement_name=CI/CD Build ${env.BUILD_NUMBER}" \
                         -F "file=@zap-report.xml"
                    """
                }
            }
        }

        stage('10. Sync SonarQube to DefectDojo') {
            steps {
                withCredentials([string(credentialsId: 'defectdojo-token', variable: 'DOJO_TOKEN')]) {
                    sh """
                    curl -X POST "${DEFECTDOJO_URL}/api/v2/import-scan/" \
                         -H "Authorization: Token \$DOJO_TOKEN" \
                         -F "active=true" \
                         -F "verified=false" \
                         -F "scan_type=SonarQube API Import" \
                         -F "product_name=HotelLux" \
                         -F "engagement_name=CI/CD Build ${env.BUILD_NUMBER}"
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Final cleanup... maintaining disk health while protecting persistent volumes."
            cleanWs() 
            // SAFE CLEAN: Removed '-a'. Keeps your base tool images on the disk.
            sh 'docker image prune -f' 
        }
        success { echo "SUCCESS: HotelLux DevSecOps Pipeline Finished!" }
        failure { echo "FAILURE: Build failed. Check the Jenkins Console Output." }
    }
}
