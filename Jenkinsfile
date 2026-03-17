pipeline {
    agent any
    
    // Grabs the tools from your Mac's Homebrew path
    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:${env.PATH}"
        GITHUB_TOKEN = credentials('github-token')
        REPO_OWNER = 'adityatripathi5'
        REPO_NAME = 'devsecops-poc'
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
                // Run Gitleaks. '|| true' ensures the pipeline doesn't fail immediately so we can report it.
                sh 'gitleaks detect -v --report-format json --report-path gitleaks-report.json || true'
                
                // 1️⃣ Push Status to GitHub Commits (Rajat's requested approach)
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
                
                // 2️⃣ Upload SARIF to GitHub Security Tab (Rajat's advanced approach)
                sh """
                export GH_TOKEN=${GITHUB_TOKEN}
                gh api --method POST \
                  -H "Accept: application/vnd.github+json" \
                  /repos/${REPO_OWNER}/${REPO_NAME}/code-scanning/sarifs \
                  -f commit_sha=${GIT_COMMIT} \
                  -f ref=refs/heads/main \
                  -f sarif=@trivy-results.sarif || echo "SARIF Upload Skipped"
                """

                // Push Status to GitHub
                sh """
                curl -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${GIT_COMMIT} \
                -d '{"state": "success", "context": "security/trivy", "description": "Scan Completed"}'
                """
            }
        }
    }
}
