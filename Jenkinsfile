pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner' [cite: 5]
        JWT_SECRET = credentials('jwt-secret-id') [cite: 5]
        DB_PASSWORD = credentials('db-password-id') [cite: 5]
        DEFECTDOJO_TOKEN = credentials('defectdojo-api-token') [cite: 5]
    }
    stages {
        stage('Code Analysis (SonarQube)') {
            steps {
                dir('backend-hotellux') { [cite: 6]
                    withSonarQubeEnv('sonar-server') { [cite: 6]
                        sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey=luxtavern-be" [cite: 6]
                    }
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') { [cite: 7]
                    waitForQualityGate abortPipeline: true [cite: 7]
                }
            }
        }
        stage('Vulnerability Scan & DefectDojo') {
            steps {
                dir('backend-hotellux') {
                    sh 'mvn dependency-check:check' [cite: 8]
                    // This requires the DefectDojo plugin in Jenkins
                    defectDojoPublisher(
                        artifact: 'target/dependency-check-report.xml', [cite: 9]
                        productName: 'Luxtavern', [cite: 9]
                        scanType: 'Dependency Check Scan', [cite: 9]
                        engagementName: 'CI/CD Pipeline' [cite: 9]
                    ) [cite: 10]
                }
            }
        }
        stage('Build & Deploy') {
            steps {
                sh 'docker-compose up -d --build' [cite: 11]
            }
        }
    }
}