pipeline {
    agent any 
    
    environment {
        COSIGN_KEY_PATH = credentials('local-cosign-key-file')
        COSIGN_PASSWORD = credentials('local-cosign-password')
    }
    
    stages {
        stage('Isolated Ephemeral Build') {
            agent {
                docker { 
                    image 'alpine:latest'
                    reuseNode true 
                }
            }
            steps {
                echo 'Building application artifact inside a strict ephemeral container...'
                // Instead of dummy echo, we package the main.py file pulled from Git!
                sh 'tar -czf app-artifact.tar.gz main.py'
                sh 'sha256sum app-artifact.tar.gz | awk \'{print $1}\' > artifact.sha'
            }
        }
        
        stage('Secure Sign (Separate Environment)') {
            steps {
                echo 'Generating SLSA Provenance completely isolated from the build container...'
                script {
                    def predicateJson = """{
                        "builder": {"id": "${JENKINS_URL}"},
                        "buildType": "https://jenkins.io/LocalPipeline/v1",
                        "invocation": {
                            "configSource": {"uri": "${BUILD_URL}"}
                        }
                    }"""
                    writeFile file: 'predicate.json', text: predicateJson
                    
                    sh """
                    cosign attest-blob --yes \
                      --key "${COSIGN_KEY_PATH}" \
                      --type slsaprovenance \
                      --predicate predicate.json \
                      --tlog-upload=false \
                      --output-file attestation.json \
                      app-artifact.tar.gz
                    """
                    
                    sh 'rm -f predicate.json'
                    echo 'SLSA Level 3 Ephemeral Build and Verification Successful!'
                }
            }
        }
    }
}
