pipeline {
    agent { label 'Jenkins' }

    environment {
        BASE_GIT_URL = "https://github.com/NIMRA345"
        GIT_CREDS   = "git-pat-creds"
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {
        stage('Build Repositories (Max 3 in Parallel)') {
            steps {
                script {
                    def repos = load 'reposConfig.groovy'
                    def batchSize = 3   // 🔥 Only 3 repos at a time

                    for (int i = 0; i < repos.size(); i += batchSize) {
                        def batch = repos[i..<Math.min(i + batchSize, repos.size())]
                        def builds = [:]

                        for (repo in batch) {
                            def repoCopy = repo

                            builds[repoCopy.name] = {
                                stage("Build ${repoCopy.name}") {
                                    dir(repoCopy.name) {
                                        deleteDir()

                                        def branchToBuild = params[repoCopy.param] ?: repoCopy.branch
                                        echo "📌 Building ${repoCopy.name} on branch: ${branchToBuild}"

                                        git branch: branchToBuild,
                                            url: "${env.BASE_GIT_URL}/${repoCopy.name}.git",
                                            credentialsId: env.GIT_CREDS

                                        sh repoCopy.cmd
                                    }
                                }
                            }
                        }

                        // Run only this batch in parallel, then wait before next batch
                        parallel builds, failFast: true
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ All repositories built successfully"
        }
        failure {
            echo "❌ One or more repositories failed"
        }
        always {
            echo "🧹 Pipeline finished"
        }
    }
}
