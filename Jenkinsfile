pipeline {
    agent any
    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'nginx:1.14.0', description: 'Docker image to scan')
    }

    environment {
        // Docker image to scan
        IMAGE_NAME = "${params.IMAGE_NAME}"
        // CI information
        REPOSITORY = "${env.JOB_NAME}"
        REF_NAME = "${env.BRANCH_NAME ?: 'main'}"
        SHA = "${env.GIT_COMMIT ?: 'local-test-' + System.currentTimeMillis()}"
    }

    stages {
        stage('Verify Environment') {
            steps {
                echo "Verifying environment..."
                sh 'docker --version'
                sh 'docker ps'
            }
        }

        stage('Pull Image') {
            steps {
                echo "Pulling Docker image"
                sh 'docker pull ${IMAGE_NAME}'
                sh 'docker images | grep nginx'
                // Get image SHA for CI event registration
                sh 'IMAGE_SHA=$(docker inspect --format=\'{{index .Id}}\' ${IMAGE_NAME})'
                sh 'echo $IMAGE_SHA'
            }

        }

        stage('Get Authentication Token') {
            steps {
                echo "Authenticating with Upwind"
                withCredentials([
                    string(credentialsId: 'upwind-client-id', variable: 'UPWIND_CLIENT_ID'),
                    string(credentialsId: 'upwind-client-secret', variable: 'UPWIND_CLIENT_SECRET')
                ]) {

                    // Use separate sh commands to avoid escape character issues
                    sh '''
                      curl -sSL -X POST \
                      --url "https://oauth.upwind.io/oauth/token" \
                      --data "audience=https://agent.upwind.io" \
                      --data "client_id=$UPWIND_CLIENT_ID" \
                      --data "client_secret=$UPWIND_CLIENT_SECRET" \
                      --data "grant_type=client_credentials" > token_response.json'''
                    // Use jq to extract out the token. Install first
                    sh 'apt-get update && apt-get install -y jq'
                    sh 'jq -r .access_token token_response.json > access_token.txt'                    
                    sh 'echo "Authentication completed"'
                }
            }
        }

        stage('Download Upwind Agent') {
            steps {
                echo "Downloading Upwind Agent"
                sh '''
                    # Read the access token
                    ACCESS_TOKEN=$(cat access_token.txt)
                    UPWIND_AGENT_URL="https://releases.upwind.io/shiftleft/stable/linux/amd64/shiftleft-linux-amd64"
                    curl -fsS -H "Authorization: Bearer $TOKEN" -L "$UPWIND_AGENT_URL" -o "./shiftleft"  
                    chmod +x "./shiftleft"                
                '''
            }
        }
        stage('Extract Docker Image') {
            steps {
                echo "Extracting Docker image for scanning"
                withCredentials([
                    string(credentialsId: 'upwind-client-id', variable: 'UPWIND_CLIENT_ID'),
                    string(credentialsId: 'upwind-client-secret', variable: 'UPWIND_CLIENT_SECRET')
                ]) {
                sh '''
                    ./shiftleft image  \
                       --initiator=upwind-demo \
                       --docker-image=${IMAGE_NAME}:latest  \
                       --docker-pull=false \
                       --upwind-client-id=upwind-client-id  \
                       --upwind-client-secret=upwind-client-secret
                '''
                }
            }
        }
        stage('Deploy') {
            steps {
                echo "Deploying image (mock step)"
                sh 'echo "This would be where your deployment steps run"'
            }
        }
    }
    post {
        always {
            node(null) {
                echo "Cleaning up workspace"
                // Archive artifacts if they exist
                archiveArtifacts artifacts: 'scan_results.json,event_response.txt,token_response.json', allowEmptyArchive: true
                // Clean workspace using deleteDir() instead of cleanWs()
                deleteDir()
            }
        }
        success {
            echo "Pipeline completed successfully"
        }
        failure {
            echo "Pipeline failed - likely due to security vulnerabilities"
        }
    }
}
