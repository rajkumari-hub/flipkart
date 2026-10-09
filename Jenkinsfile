
pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Select environment'
        )

        choice(
            name: 'CONFIG_TYPE',
            choices: ['both', 'environment', 'node'],
            description: 'Select configuration type'
        )

        string(
            name: 'INSTANCE_TYPE',
            defaultValue: '',
            description: 'Mandatory for node updates'
        )

        string(
            name: 'K8S_VERSION',
            defaultValue: '',
            description: 'Optional: Kubernetes version'
        )

        string(
            name: 'CPU',
            defaultValue: '',
            description: 'Optional: CPU value'
        )

        string(
            name: 'MEMORY',
            defaultValue: '',
            description: 'Optional: Memory value'
        )
    }

    stages {
        stage('Display Parameters') {
            steps {
                echo "Environment: ${params.ENVIRONMENT}"
                echo "Config Type: ${params.CONFIG_TYPE}"
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    if (!params.ENVIRONMENT?.trim()) {
                        error('ENVIRONMENT is mandatory')
                    }

                    if (!params.CONFIG_TYPE?.trim()) {
                        error('CONFIG_TYPE is mandatory')
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        if (!params.INSTANCE_TYPE?.trim()) {
                            error('INSTANCE_TYPE is mandatory for node updates')
                        }
                    }

                    echo 'All mandatory parameters validated successfully'
                }
            }
        }

        stage('Read Configuration Files') {
            steps {
                script {
                    def types = params.CONFIG_TYPE == 'both'
                        ? ['environment', 'node']
                        : [params.CONFIG_TYPE]

                    types.each { type ->
                        def path = "${type}/${params.ENVIRONMENT}.json"

                        if (!fileExists(path)) {
                            error("Configuration file missing: ${path}")
                        }

                        def config = readJSON file: path, returnPojo: true

                        if (!(config instanceof Map)) {
                            error("Expected a JSON object in ${path}")
                        }

                        echo "Validated JSON file: ${path}"
                    }
                }
            }
        }

        stage('Update JSON Configuration') {
            steps {
                script {
                    def types = params.CONFIG_TYPE == 'both'
                        ? ['environment', 'node']
                        : [params.CONFIG_TYPE]

                    types.each { type ->
                        def path = "${type}/${params.ENVIRONMENT}.json"

                        def config = readJSON file: path, returnPojo: true

                        if (type == 'node') {
                            config.instanceType = params.INSTANCE_TYPE.trim()

                            if (params.K8S_VERSION?.trim()) {
                                config.kubernetes.version =
                                    params.K8S_VERSION.trim()
                            }

                            if (params.CPU?.trim()) {
                                config.resources.cpu = params.CPU.trim()
                            }

                            if (params.MEMORY?.trim()) {
                                config.resources.memory =
                                    params.MEMORY.trim()
                            }

                            writeJSON file: path,
                                      json: config,
                                      pretty: 4

                            echo "Updated local configuration: ${path}"
                        }
                    }
                }
            }
        }

        stage('Verify Updated JSON') {
            steps {
                script {
                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def path = "node/${params.ENVIRONMENT}.json"

                        def config = readJSON file: path, returnPojo: true

                        echo "Updated Instance Type: ${config.instanceType}"
                        echo "Updated Kubernetes Version: ${config.kubernetes.version}"
                        echo "Updated CPU: ${config.resources.cpu}"
                        echo "Updated Memory: ${config.resources.memory}"
                    }
                }
            }
       }
        stage('Create Git Feature Branch') {
    steps {
        script {
            def branchName =
                "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"

            env.FEATURE_BRANCH = branchName

            sh """
                git config user.name "Jenkins Automation"
                git config user.email "jenkins-automation@example.com"

                git switch -c "${branchName}"

                git add environment/*.json node/*.json

                if git diff --cached --quiet; then
                    echo "No configuration changes detected"
                else
                    git commit -m "Update ${params.ENVIRONMENT} configuration"
                fi
            """

            echo "Feature branch prepared: ${branchName}"
        }
    }
}
        stage('Push Feature Branch') {
    steps {
        withCredentials([
            gitUsernamePassword(
                credentialsId: 'github-private-creds',
                gitToolName: 'Default'
            )
        ]) {
            sh 'git push -u origin "$FEATURE_BRANCH"'
        }
    }
}
    }
    
}
