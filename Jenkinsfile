pipeline {
    agent any

    parameters {
        choice(
            name: 'ENV_BRANCH',
            choices: ['develop', 'test', 'staging', 'prod'],
            description: 'Select the fallback branch to build if repo-specific tag not set'
        )
    }

    options {
        timestamps()
        ansiColor('xterm')
        timeout(time: 4, unit: 'HOURS')
    }

    environment {
        GIT_BASE_URL = "https://github.com/NIMRA345"
        GIT_CREDS = "github-pat"  // Jenkins Credentials ID
        REPO_ENV_FILE = "${WORKSPACE}/build.env"
    }

    stages {
        stage('Load .env') {
            steps {
                script {
                    def props = readProperties file: "${env.REPO_ENV_FILE}"
                    props.each { k, v ->
                        env[k] = v
                    }
                    echo "✅ Loaded environment variables from .env"
                }
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh "mkdir -p ${env.BUILD_HOME}"
            }
        }

        stage('Parse Repositories') {
            steps {
                script {
                    repos = env.REPO_LIST.split(',').collect { it.trim() }
                }
            }
        }

        stage('Clone & Build Repositories') {
            steps {
                script {
                    def batches = repos.collate(env.MAX_PARALLEL.toInteger())

                    for (batch in batches) {
                        def parallelStages = [:]

                        batch.each { repoName ->
                            def name = repoName
                            parallelStages[name] = {
                                node {
                                    timeout(time: 1, unit: 'HOURS') {
                                        dir("${env.BUILD_HOME}/${name}") {
                                            deleteDir()
                                            try {
                                                echo "======================================================="
                                                echo "🔽 Cloning $name"

                                                // Determine branch/tag
                                                def branch = env["${name.toUpperCase().replaceAll('-', '_')}_TAG"] ?: params.ENV_BRANCH ?: "develop"
                                                echo "✅ Using branch/tag for ${name}: ${branch}"

                                                // Clone with Jenkins credentials
                                                withCredentials([usernamePassword(
                                                    credentialsId: env.GIT_CREDS,
                                                    usernameVariable: 'GIT_USER',
                                                    passwordVariable: 'GIT_TOKEN'
                                                )]) {
                                                    sh """
                                                        git clone -b ${branch} https://${GIT_USER}:${GIT_TOKEN}@github.com/NIMRA345/${name}.git . || exit 1
                                                    """
                                                }

                                                // Build detection
                                                if (fileExists('pom.xml')) {
                                                    def profiles = env["${name.toUpperCase().replaceAll('-', '_')}_PROFILES"] ?: "!tag"
                                                    echo "☕ Maven project detected. Profiles: ${profiles}"
                                                    sh "mvn -B -U -P '${profiles}' -DskipTests clean install"
                                                } else if (fileExists('build.sh')) {
                                                    echo "⚡ Special build.sh detected, executing..."
                                                    sh "chmod +x build.sh && ./build.sh"
                                                } else if (fileExists('Makefile')) {
                                                    echo "⚡ Makefile detected, running make..."
                                                    sh "make"
                                                } else {
                                                    echo "⏭ No recognized build file found, skipping ${name}"
                                                }

                                                echo "✅ Build finished for ${name}"
                                                echo "======================================================="
                                            } catch (e) {
                                                error("❌ Build failed for ${name}: ${e}")
                                            }
                                        }
                                    }
                                }
                            }
                        }

                        parallel parallelStages
                    }
                }
            }
        }
    }

    post
        always {
            echo "🎯 Pipeline execution completed at ${new Date().format("yyyy-MM-dd HH:mm:ss")}"
        }
    }
}
