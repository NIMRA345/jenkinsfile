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
        stage('Build Repositories in Parallel') {
            steps {
                script {
                    def repos = load 'reposConfig.groovy'
                    def builds = [:]

                    for (repo in repos) {
                        def repoName = repo.name
                        def repoCfg  = repo

                        builds[repoName] = {
                            stage("Build ${repoName}") {
                                dir(repoName) {
                                    deleteDir()

                                    def branchToBuild = params[repoCfg.param] ?: repoCfg.branch
                                    echo "📌 Building ${repoName} on branch: ${branchToBuild}"

                                    git branch: branchToBuild,
                                        url: "${env.BASE_GIT_URL}/${repoName}.git",
                                        credentialsId: env.GIT_CREDS

                                    sh repoCfg.cmd
                                }
                            }
                        }
                    }

                    parallel builds, failFast: true
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
