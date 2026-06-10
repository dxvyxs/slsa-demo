pipeline {
    agent any

    tools {
        // References the tool name you set in Step 2
        'org.jenkinsci.plugins.DependencyCheck.tools.DependencyCheckInstallation' 'OWASP-DC'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Dependency Scan') {
            steps {
                dependencyCheck(
                    additionalArguments: '''
                        --scan ./
                        --format HTML
                        --format XML
                        --out ./dependency-check-report
                        --prettyPrint
                    ''',
                    odcInstallation: 'OWASP-DC'
                )
            }
        }

        stage('Scorecard') {
            steps {
                dependencyCheckPublisher(
                    pattern: 'dependency-check-report/dependency-check-report.xml',
                    failedTotalCritical: 0,    // Fail build if ANY critical vuln found
                    unstableTotalHigh: 5,       // Mark unstable if > 5 high vulns
                    unstableTotalMedium: 10
                )
            }
        }

    }

    post {
        always {
            // Publish the HTML scorecard report
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'dependency-check-report',
                reportFiles: 'dependency-check-report.html',
                reportName: '🔐 Dependency Scorecard'
            ])
        }
    }
}
