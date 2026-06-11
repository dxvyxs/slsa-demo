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
                          --repo=github.com/abluva-research/policy-sdk \
                          --format default
                    '''
                }
            }
        }

        stage('Sign Release') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        curl -sL https://github.com/cli/cli/releases/download/v2.49.2/gh_2.49.2_linux_amd64.tar.gz -o gh.tar.gz
                        tar -xzf gh.tar.gz
                        chmod +x gh_2.49.2_linux_amd64/bin/gh

                        cosign sign-blob \
                          --yes \
                          --output-signature release.sig \
                          --output-certificate release.pem \
                          Jenkinsfile

                        ./gh_2.49.2_linux_amd64/bin/gh release upload v1.0.0 release.sig release.pem \
                          --repo abluva-research/policy-sdk \
                          --clobber
                    '''
                }
            }
        }
    }
}
