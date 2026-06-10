pipeline {
    agent any

    stages {
        stage('OpenSSF Scorecard') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        curl -sLO https://github.com/ossf/scorecard/releases/latest/download/scorecard_linux_amd64.tar.gz
                        tar -xzf scorecard_linux_amd64.tar.gz
                        chmod +x scorecard_linux_amd64

                        ./scorecard_linux_amd64 \
                          --repo=github.com/dxvyxs/slsa-demo \
                          --format default
                    '''
                }
            }
        }
    }
}
