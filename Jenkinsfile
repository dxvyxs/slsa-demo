@Library('github-issue-creator') _

pipeline {
    agent any
    
    environment {
        GITHUB_REPO = 'dxvyxs/slsa-demo'
        GITHUB_TOKEN = credentials('github-issue-token')
        GITHUB_ISSUE_TOKEN = credentials('github-issue-token')
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

                    docker run --rm \
                        -v "$(pwd):/workspace" \
                        anchore/syft /workspace \
                        -o cyclonedx-json=/workspace/sbom.json

                    echo "Checking SBOM..."
                    ls -lh sbom.json

                    echo "Scanning SBOM with Grype..."

                    docker run --rm \
                        -v "$(pwd):/workspace" \
                        anchore/grype sbom:/workspace/sbom.json \
                        --output sarif \
                        > grype-report.sarif

                    echo "Checking Grype report..."
                    ls -lh grype-report.sarif
                '''
            }
        }
        
        stage('Secret Scan') {
            steps {
                sh '''
                    echo "Scanning for secrets with TruffleHog..."
                    docker run -v $(pwd):/workspace trufflesecurity/trufflehog filesystem /workspace \
                        --sarif > trufflehog-report.sarif || true
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

        stage('Build') {
            steps {
                sh '''
                    echo "Building image using docker compose..."
                    export IMAGE_NAME=scim
                    export IMAGE_TAG=latest
                    docker compose -f scim/docker/docker-compose.yml build
                    echo "Saving image to current directory..."
                    docker save -o scim.tar docker-scim:latest
                '''
            }
        }

        stage('Container Image Scan') {
            steps {
                sh '''
                    echo "Scanning container image with Trivy (SARIF output)..."
                    docker run --rm \
                        -v $(pwd):/workspace \
                        aquasec/trivy:0.69.3 image \
                        --input /workspace/scim.tar \
                        --scanners vuln,secret,misconfig \
                        --format sarif \
                        -o /workspace/trivy-container-report.sarif
                '''
            }
        }
        
        stage('Create GitHub Issues from SARIF') {
            steps {
                script {
                    def sarifFiles = [
                        'grype-report.sarif',
                        'trufflehog-report.sarif',
                        'trivy-iac-report.sarif',
                        'trivy-container-report.sarif'
                    ]
                    
                    sarifFiles.each { sarifFile ->
                        if (fileExists(sarifFile)) {
                            echo "Processing ${sarifFile}..."
                            def report = readJSON file: sarifFile, returnPojo: true
                            
                            report.runs?.each { run ->
                                def toolName = run.tool.driver.name ?: 'unknown'
                                def resultsCount = run.results?.size() ?: 0
                                echo "Found ${resultsCount} results from ${toolName}"
                                
                                run.results?.each { result ->
                                    def ruleId = result.ruleId ?: result.rule?.id ?: 'unknown'
                                    def location = result.locations?.getAt(0)?.physicalLocation?.artifactLocation?.uri ?: 'unknown'
                                    def findingKey = "${toolName}|${ruleId}|${location}".replaceAll(/[^a-zA-Z0-9_-]/, '-')
                                    
                                    notifyGithubIssue(
                                        githubRepo: 'dxvyxs/slsa-demo',
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
            echo 'Pipeline completed.'
        }
    }
}
