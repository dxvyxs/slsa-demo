pipeline {
    agent any

    stages {
        stage('OpenSSF Scorecard') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        curl -sL https://github.com/ossf/scorecard/releases/download/v5.1.1/scorecard_5.1.1_linux_amd64.tar.gz -o scorecard.tar.gz
                        tar -xzf scorecard.tar.gz
                        chmod +x scorecard

                        ./scorecard \
                          --repo=github.com/dxvyxs/slsa-demo \
                          --format default
                    '''
                }
            }
        }
    }
}
