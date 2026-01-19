pipeline {
    agent { label 'Jenkins' }

    parameters {
        string(name: 'IE_GLOBAL_TAG', defaultValue: 'develop', description: 'Branch/tag for ie-global')
        string(name: 'IE_DEPS_TAG', defaultValue: 'develop', description: 'Branch/tag for ie-deps')
        string(name: 'DB_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-tch-database')
        string(name: 'BO_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-tch-backoffice')
        string(name: 'MAPS_SERVICE_CONFIG_TAG', defaultValue: 'develop', description: 'Branch/tag for maps-service-config')
        string(name: 'IPS_BO_API_SPECS_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-bo-api-specs')
        string(name: 'OPSUI_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-tch-opsui')
        string(name: 'SWITCH_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-tch-switch')
        string(name: 'LIABILITY_MANAGER_TAG', defaultValue: 'develop', description: 'Branch/tag for liability-manager')
        string(name: 'MAPS_TCH_TAG', defaultValue: 'develop', description: 'Branch/tag for maps-tch')
        string(name: 'MAPS_CORE_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-maps-core')
        string(name: 'IPS_TRANSFORMER_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-tch-transformer')
        string(name: 'IPS_MESSAGES_ISO20022_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-messages-iso20022')
        string(name: 'IPS_MESSAGES_CANONICAL_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-messages-canonical')
        string(name: 'IPS_DSP_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-dsp')
        string(name: 'RAMBUS_STE_CLIENT_TAG', defaultValue: 'develop', description: 'Branch/tag for rambus-ste-client')
        string(name: 'MAPS_DSP_TAG', defaultValue: 'develop', description: 'Branch/tag for maps-dsp')
        string(name: 'LEGACY_DSP_TAG', defaultValue: 'develop', description: 'Branch/tag for legacy-dsp')
        string(name: 'SACHSIM_UI_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-tch-msimui')
        string(name: 'LEGACY_TRANSFORMER_TAG', defaultValue: 'develop', description: 'Branch/tag for ips-transformer')
        string(name: 'IS020022_TCH_VERSION_TAG', defaultValue: 'develop', description: 'Branch/tag for IS020022_TCH_VERSION')
    }

    environment {
        BASE_GIT_URL = "https://github.com/NIMRA345"
        GIT_CREDS    = "git-pat-creds"
        BUILD_HOME   = "/home/kenneth_gentry/workspace/ado"
    }

    stages {
        stage('Build All Repositories in Parallel') {
            steps {
                script {
                    // Load repos config
                    def repos = load 'reposConfig.groovy'

                    def parallelBuilds = [:]

                    for (repo in repos) {
                        def repoCopy = repo // closure binding
                        parallelBuilds[repoCopy.name] = {
                            dir("${env.BUILD_HOME}/${repoCopy.name}") {
                                deleteDir()
                                git branch: params[repoCopy.param] ?: repoCopy.branch,
                                    url: "${env.BASE_GIT_URL}/${repoCopy.name}.git",
                                    credentialsId: env.GIT_CREDS

                                echo "📌 Building ${repoCopy.name} on ${params[repoCopy.param] ?: repoCopy.branch}"
                                sh "${repoCopy.cmd.replace('!tag', params[repoCopy.param] ?: repoCopy.branch)}"
                            }
                        }
                    }

                    parallel parallelBuilds
                }
            }
        }
    }

    post {
        success { echo "✅ All repositories built successfully" }
        failure { echo "❌ One or more builds failed" }
    }
}
