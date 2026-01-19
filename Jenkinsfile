pipeline {
    agent any

    parameters {
        string(
            name: 'BRANCH_MAP',
            defaultValue: '{"ie-global":"develop","ips-dsp":"develop"}',
            description: 'JSON map of repo -> branch. Example: {"repo1":"develop","repo2":"staging"}'
        )
    }

    options {
        timestamps()
        ansiColor('xterm')
        timeout(time: 4, unit: 'HOURS')
    }

    environment {
        GIT_BASE_URL = "https://github.com/NIMRA345"
        GIT_CREDS = "github-pat"
        REPO_ENV_FILE = "${WORKSPACE}/build.env"
    }

    stages {

        stage('Load .env') {
            steps {
                script {
                    if (fileExists(env.REPO_ENV_FILE)) {
                        def props = readProperties file: "${env.REPO_ENV_FILE}"
                        props.each { k, v -> env[k] = v }
                        echo "✅ Loaded environment variables from .env"
                    } else {
                        error("❌ build.env file not found at ${env.REPO_ENV_FILE}")
                    }
                }
            }
        }

        stage('Prepare Workspace') {
            steps {
                cleanWs()
                sh "mkdir -p ${env.BUILD_HOME}"
            }
        }

        stage('Parse Repositories') {
            steps {
                script {
                    repos = env.REPO_LIST.split(',').collect { it.trim() }
                    branchMap = readJSON text: params.BRANCH_MAP
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

                                                // Determine branch: JSON input > .env > default 'develop'
                                                def branch = branchMap[name] ?: 
                                                    env["${name.toUpperCase().replaceAll('-', '_')}_TAG"] ?: 
                                                    "develop"

                                                echo "✅ Using branch for ${name}: ${branch}"

                                                // 🔐 Secure clone using GitHub Username + PAT
                                                withCredentials([
                                                    usernamePassword(
                                                        credentialsId: env.GIT_CREDS,
                                                        usernameVariable: 'GIT_USER',
                                                        passwordVariable: 'GIT_TOKEN'
                                                    )
                                                ]) {
                                                    sh """
                                                        echo "🔽 Cloning ${name} from branch ${branch}"
                                                        git clone --branch ${branch} --depth 1 \
                                                        https://${GIT_USER}:${GIT_TOKEN}@github.com/NIMRA345/${name}.git .
                                                    """
                                                }

                                                // Detect and build
                                                if (fileExists('pom.xml')) {
                                                    def profiles = env["${name.toUpperCase().replaceAll('-', '_')}_PROFILES"] ?: "!tag"
                                                    echo "☕ Maven project detected. Profiles: ${profiles}"
                                                    sh "mvn -B -U -P ${profiles} -DskipTests clean install"
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
                                                echo "❌ Build failed for ${name}: ${e}"
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

    post {
        always {
            echo "🎯 Pipeline execution completed at ${new Date().format('yyyy-MM-dd HH:mm:ss')}"
        }
    }
}
