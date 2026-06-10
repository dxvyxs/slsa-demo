pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Dependency Scan') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                    dependencyCheck(
                        additionalArguments: """
                            --scan ./
                            --format HTML
                            --format XML
                            --out ./dependency-check-report
                            --prettyPrint
                            --nvdApiKey ${NVD_KEY}
                        """,
                        odcInstallation: 'OWASP-DC'
                    )
                }
            }
        }

        stage('Scorecard') {
            steps {
                dependencyCheckPublisher(
                    pattern: 'dependency-check-report/dependency-check-report.xml',
                    failedTotalCritical: 0,
                    unstableTotalHigh: 5,
                    unstableTotalMedium: 10
                )
            }
        }

    }

    post {
        always {
            publishHTML(target: [
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'dependency-check-report',
                reportFiles: 'dependency-check-report.html',
                reportName: '🔐 Dependency Scorecard'
            ])
        }
    }
}
