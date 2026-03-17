pipeline {
    agent any
    
    // Grabs the tools from your Mac's Homebrew path
    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:${env.PATH}"
        GITHUB_TOKEN = credentials('github-token')
        REPO_OWNER = 'adityatripathi5'
        REPO_NAME = 'devsecops-poc'
        // Forces Trivy to ignore Mac Docker settings so it can download its database
        DOCKER_CONFIG = '/tmp'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    // Get the commit hash to update GitHub status
                    env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                }
            }
        }

        stage('Gitleaks Scan') {
            steps {
                // Run Gitleaks
                sh 'gitleaks detect -v --report-format json --report-path gitleaks-report.json || true'
                
                // 1️⃣ Push Status to GitHub Commits
                sh """
                curl -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                -H "Accept: application/vnd.github.v3+json" \
                https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${GIT_COMMIT} \
                -d '{"state": "success", "context": "security/gitleaks", "description": "Scan Completed"}'
                """
            }
        }

        stage('Trivy Scan & SARIF Upload') {
            steps {
                // Run Trivy and generate a SARIF report
                sh 'trivy fs . --format sarif --output trivy-results.sarif || true'
                
                // 2️⃣ Upload SARIF to GitHub Security Tab (Bulletproof API method)
                sh """
                if [ -f "trivy-results.sarif" ]; then
                    # Compress and encode to Base64
                    gzip -c trivy-results.sarif | base64 | tr -d '\\n' > payload.b64
                    
                    # Safely build the exact JSON payload GitHub expects without escaping errors
                    echo '{"commit_sha":"${GIT_COMMIT}","ref":"refs/heads/main","sarif":"' > request.json
                    cat payload.b64 >> request.json
                    echo '"}' >> request.json
                    
                    # Push to GitHub API
                    curl -s -w "\\nHTTP_STATUS:%{http_code}\\n" -X POST \
                      -H "Authorization: Bearer ${GITHUB_TOKEN}" \
                      -H "Accept: application/vnd.github.v3+json" \
                      https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/code-scanning/sarifs \
                      -d @request.json
                else
                    echo "Trivy failed to generate SARIF."
                fi
                """

                // 3️⃣ Push Commit Status to GitHub
                sh """
                curl -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${GIT_COMMIT} \
                -d '{"state": "
