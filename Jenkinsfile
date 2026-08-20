@Library('github-issue-creator') _

pipeline {
    agent any
    
    environment {
        GITHUB_REPO = 'dxvyxs/slsa-demo'
        GITHUB_TOKEN = credentials('github-issues')
        GITHUB_ISSUE_TOKEN = credentials('github-issues')
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/dxvyxs/slsa-demo',
                        credentialsId: 'github-issue-token'
                    ]]
                )
            }
        }
        
        stage('SBOM & Vulnerability Scan') {
            steps {
                sh '''
                    echo "Generating SBOM with Syft..."
                    docker run -v $(pwd):/workspace anchore/syft /workspace -o cyclonedx-json > sbom.json
                    
                    echo "Scanning SBOM with Grype (SARIF output)..."
                    docker run --rm \
                        -v $(pwd):/workspace \
                        anchore/grype sbom:/workspace/sbom.json \
                        --output sarif > grype-report.sarif
                '''
            }
        }
        
        stage('Secret Scan') {
            steps {
                sh '''
                    echo "Scanning for secrets with TruffleHog..."
                    docker run -v $(pwd):/workspace trufflesecurity/trufflehog filesystem /workspace \
                        --json > trufflehog-raw.json || true
                    
                    echo "Converting TruffleHog to SARIF..."
                    docker run -v $(pwd):/workspace trufflesecurity/trufflehog filesystem /workspace \
                        --sarif > trufflehog-report.sarif || true
                    
                    echo "Scanning with Gitleaks..."
                    docker run -v $(pwd):/workspace zricethezav/gitleaks:latest detect \
                        --source /workspace \
                        --report-format sarif \
                        --report-path /workspace/gitleaks-report.sarif || true
                '''
            }
        }
        
        stage('IaC Scan') {
            steps {
                sh '''
                    echo "Scanning IaC with Trivy (SARIF output)..."
                    docker run --rm -v $(pwd):/workspace \
                        aquasec/trivy:0.69.3 fs /workspace \
                        --scanners misconfig,vuln,secret \
                        --format sarif \
                        -o /workspace/trivy-iac-report.sarif
                '''
            }
        }
        
        stage('Create GitHub Issues from SARIF') {
            steps {
                script {
                    
                    
                    def sarifFiles = [
                        'grype-report.sarif',
                        'trufflehog-report.sarif',
                        'gitleaks-report.sarif',
                        'trivy-iac-report.sarif'
                    ]
                    
                    sarifFiles.each { sarifFile ->
                        if (fileExists(sarifFile)) {
                            echo "Processing ${sarifFile}..."
                            def report = new groovy.json.JsonSlurper().parseText(readFile(sarifFile))
                            
                            report.runs?.each { run ->
                                def toolName = run.tool.driver.name ?: 'unknown'
                                def resultsCount = run.results?.size() ?: 0
                                echo "Found ${resultsCount} results from ${toolName}"
                                
                                run.results?.each { result ->
                                    def ruleId = result.ruleId ?: result.rule?.id ?: 'unknown'
                                    def location = result.locations?.getAt(0)?.physicalLocation?.artifactLocation?.uri ?: 'unknown'
                                    def findingKey = "${toolName}|${ruleId}|${location}".replaceAll(/[^a-zA-Z0-9_-]/, '-')
                                    
                                    notifyGithubIssue(
                                        githubRepo: 'abluva-research/mcp-trust-plane',
                                        credentialId: 'github-issue-token',
                                        customFailureKey: findingKey,
                                        labels: [toolName, result.level ?: 'warning', 'automated-scan']
                                    )
                                }
                            }
                        } else {
                            echo "WARNING: ${sarifFile} not found"
                        }
                    }
                }
            }
        }
        
        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: '*.sarif, *.json', fingerprint: true
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
