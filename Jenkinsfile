is it crt? pipeline {
    agent any

    parameters {
        string(
            name: 'REPO_LIST',
            defaultValue: 'ie-global,ie-deps,ips-bo-api-specs,ips-tch-backoffice,ips-dsp,ips-messages-canonical,ips-messages-iso20022,maps-service-config,ips-maps-core,ips-tch-database,ips-tch-msimui,ips-tch-transformer,ips-transformer,maps-tch,rambus-ste-client',
            description: 'Comma separated GitHub repository names'
        )

        choice(
            name: 'ENV_BRANCH',
            choices: ['develop', 'test', 'staging', 'prod'],
            description: 'Select the environment branch to build'
        )
    }

    options {
        timestamps()
        ansiColor('xterm')
        timeout(time: 4, unit: 'HOURS')
    }

    environment {
        // Git settings
        GIT_BASE_URL = "https://github.com/NIMRA345"
        GIT_CREDS = "github-pat"

        // Workspace & parallelism
        BUILD_HOME = "/home/kenneth_gentry/workspace/ado"
        MAX_PARALLEL = "5"

        // Repo-specific tags
        IE_GLOBAL_TAG = "develop"
        IE_DEPS_TAG = "develop"
        DB_TAG = "6.22.6"
        BO_TAG = "6.81.1"
        MAPS_SERVICE_CONFIG_TAG = "6.17.0"
        IPS_BO_API_SPECS_TAG = "2.8.0"
        OPSUI_TAG = "6.31.1"
        SWITCH_TAG = "7.67.18"
        LIABILITY_MANAGER_TAG = "7.67.18"
        MAPS_TCH_TAG = "8.44.1"
        MAPS_CORE_TAG = "31.20.0"
        IPS_TRANSFORMER_TAG = "8.14.0"
        IPS_MESSAGES_ISO20022_TAG = "6.25.0"
        IPS_MESSAGES_CANONICAL_TAG = "9.5.0"
        IPS_DSP_TAG = "4.6.0-180"
        RAMBUS_STE_CLIENT_TAG = "0.1.9-41"
        MAPS_DSP_TAG = "7.12.0"
        LEGACY_DSP_TAG = "6.81.1"
        SACHSIM_UI_TAG = "6.24.0"
        LEGACY_TRANSFORMER_TAG = "6.27.0"
        IS020022_TCH_VERSION_TAG = "6.25.0"

        // Maven profiles per repo
        IPS_TCH_BACKOFFICE_PROFILES = "!tag,rpm,war"
        IPS_DSP_PROFILES = "!tag,rpm"
        IPS_TCH_TRANSFORMER_PROFILES = "!tag,rpm,pre-prod"
        IPS_TCH_DATABASE_PROFILES = "rpm,war,assembly-db2,txn-jooq,ana-jooq,test-suites"
        IPS_MAPS_CORE_PROFILES = "!tag,rpm"
        IPS_TCH_MSIMUI_PROFILES = "!tag,rpm"
    }

    stages {
        stage('Prepare Workspace') {
            steps {
                sh "mkdir -p ${BUILD_HOME}"
            }
        }

        stage('Parse Repositories') {
            steps {
                script {
                    repos = params.REPO_LIST.split(',').collect { it.trim() }
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
                                        dir("${BUILD_HOME}/${name}") {
                                            deleteDir()
                                            try {
                                                echo "======================================================="
                                                echo "🔽 Cloning $name"

                                                // Fix for hyphenated repo names
                                                def branch = env["${name.toUpperCase().replaceAll('-', '_')}_TAG"] ?: params.ENV_BRANCH ?: "develop"
                                                echo "✅ Using branch/tag for ${name}: ${branch}"

                                                // Clone with credentials safely
                                                withCredentials([usernamePassword(
                                                    credentialsId: env.GIT_CREDS,
                                                    usernameVariable: 'GIT_USER',
                                                    passwordVariable: 'GIT_TOKEN'
                                                )]) {
                                                    sh """
                                                        git clone -b ${branch} https://${GIT_USER}:${GIT_TOKEN}@github.com/nimra/${name}.git . || exit 1
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

    post {
        always {
            echo "🎯 Pipeline execution completed at ${new Date().format("yyyy-MM-dd HH:mm:ss")}"
        }
    }
}