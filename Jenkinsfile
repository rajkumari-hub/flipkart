pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'stage', 'uat', 'prod'],
               description: 'Environment to update')

        choice(name: 'CONFIG_TYPE', choices: ['both', 'environment', 'node'],
               description: 'Configuration to update')

        string(name: 'INSTANCE_TYPE', defaultValue: '',
               description: 'Required for node or both updates')
        string(name: 'K8S_VERSION', defaultValue: '',
               description: 'Optional Kubernetes version')
        string(name: 'CPU', defaultValue: '', description: 'Optional CPU')
        string(name: 'MEMORY', defaultValue: '', description: 'Optional memory')

        string(name: 'ENV_APPNAME', defaultValue: '',
               description: 'Optional application name')
        string(name: 'ENV_VERSION', defaultValue: '',
               description: 'Optional new application version')
        string(name: 'ENV_REPLICAS', defaultValue: '',
               description: 'Optional positive replica count')
        string(name: 'ENV_LOGLEVEL', defaultValue: '',
               description: 'DEBUG, INFO, WARN, or ERROR')

        string(name: 'NODE_NAME', defaultValue: '', description: 'Optional node name')
        string(name: 'NODE_TYPE', defaultValue: '', description: 'Optional node type')
        string(name: 'NODE_REGION', defaultValue: '', description: 'Optional AWS region')
        string(name: 'NODE_AZ', defaultValue: '', description: 'Optional availability zone')
    }

    environment {
        GIT_REPO = 'https://github.com/asifshaik6558/flipkart.git'
        GITHUB_REPO = 'asifshaik6558/flipkart'
        BASE_BRANCH = 'main'
        SONAR_SERVER = 'SonarQube-Server'
        SONAR_SCANNER = 'SonarQube-Scanner'
        NEXUS_REPOSITORY_URL = 'http://localhost:8081/repository/flipkart-config-releases'
        CONFIG_CHANGES = 'false'
    }

    stages {
        stage('Checkout GitHub Repository') {
            steps {
                deleteDir()

                git branch: 'main',
                    credentialsId: 'git',
                    url: 'https://github.com/asifshaik6558/flipkart.git'

                sh '''
                    set -eu
                    git log -1 --oneline
                    find environment node -maxdepth 1 -type f -name '*.json' | sort
                '''
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    if (!(params.ENVIRONMENT in ['dev', 'stage', 'uat', 'prod'])) {
                        error('Invalid environment')
                    }

                    if (!(params.CONFIG_TYPE in ['both', 'environment', 'node'])) {
                        error('Invalid configuration type')
                    }

                    if (params.CONFIG_TYPE in ['node', 'both'] &&
                        !params.INSTANCE_TYPE?.trim()) {
                        error('INSTANCE_TYPE is required for node updates')
                    }

                    if (params.ENV_REPLICAS?.trim() &&
                        !(params.ENV_REPLICAS.trim() ==~ /[1-9][0-9]*/)) {
                        error('ENV_REPLICAS must be a positive integer')
                    }

                    if (params.ENV_LOGLEVEL?.trim() &&
                        !(params.ENV_LOGLEVEL.trim() in ['DEBUG', 'INFO', 'WARN', 'ERROR'])) {
                        error('Invalid ENV_LOGLEVEL')
                    }

                    echo 'Parameter validation passed'
                }
            }
        }

        stage('Read and Validate JSON') {
            steps {
                script {
                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        def f = "environment/${params.ENVIRONMENT}.json"

                        if (!fileExists(f)) {
                            error("Missing file: ${f}")
                        }

                        def c = readJSON(file: f, returnPojo: true)

                        if (!(c instanceof Map)) {
                            error("Invalid JSON object: ${f}")
                        }
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def f = "node/${params.ENVIRONMENT}.json"

                        if (!fileExists(f)) {
                            error("Missing file: ${f}")
                        }

                        def c = readJSON(file: f, returnPojo: true)

                        if (!(c instanceof Map)) {
                            error("Invalid JSON object: ${f}")
                        }
                    }
                }
            }
        }

        stage('Update JSON Configuration') {
            steps {
                script {
                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        def f = "environment/${params.ENVIRONMENT}.json"
                        def c = readJSON(file: f, returnPojo: true)

                        if (params.ENV_VERSION?.trim()) {
                            if (params.ENV_VERSION.trim() == c.version?.toString()) {
                                error('New version must differ from current version')
                            }

                            c.version = params.ENV_VERSION.trim()
                        }

                        if (params.ENV_APPNAME?.trim()) {
                            c.appName = params.ENV_APPNAME.trim()
                        }

                        if (params.ENV_REPLICAS?.trim()) {
                            c.replicas = params.ENV_REPLICAS.trim().toInteger()
                        }

                        if (params.ENV_LOGLEVEL?.trim()) {
                            c.logLevel = params.ENV_LOGLEVEL.trim()
                        }

                        writeJSON(file: f, json: c, pretty: 4)
                        echo "Updated ${f}"
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def f = "node/${params.ENVIRONMENT}.json"
                        def c = readJSON(file: f, returnPojo: true)

                        c.instanceType = params.INSTANCE_TYPE.trim()

                        if (params.NODE_NAME?.trim()) {
                            c.nodeName = params.NODE_NAME.trim()
                        }

                        if (params.NODE_TYPE?.trim()) {
                            c.nodeType = params.NODE_TYPE.trim()
                        }

                        if (params.NODE_REGION?.trim()) {
                            c.region = params.NODE_REGION.trim()
                        }

                        if (params.NODE_AZ?.trim()) {
                            c.availabilityZone = params.NODE_AZ.trim()
                        }

                        if (params.K8S_VERSION?.trim()) {
                            if (!(c.kubernetes instanceof Map)) {
                                c.kubernetes = [:]
                            }

                            c.kubernetes.version = params.K8S_VERSION.trim()
                        }

                        if (params.CPU?.trim() || params.MEMORY?.trim()) {
                            if (!(c.resources instanceof Map)) {
                                c.resources = [:]
                            }

                            if (params.CPU?.trim()) {
                                c.resources.cpu = params.CPU.trim()
                            }

                            if (params.MEMORY?.trim()) {
                                c.resources.memory = params.MEMORY.trim()
                            }
                        }

                        writeJSON(file: f, json: c, pretty: 4)
                        echo "Updated ${f}"
                    }
                }
            }
        }

        stage('Verify JSON and Diff') {
            steps {
                sh '''
                    set -eu
                    git diff --check
                    echo "===== Selected configuration diff ====="
                    git diff -- environment node
                '''

                script {
                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        def c = readJSON(
                            file: "environment/${params.ENVIRONMENT}.json",
                            returnPojo: true
                        )

                        echo "Environment: ${c.environment}, app: ${c.appName}, version: ${c.version}"
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def c = readJSON(
                            file: "node/${params.ENVIRONMENT}.json",
                            returnPojo: true
                        )

                        echo "Node: ${c.nodeName}, instance type: ${c.instanceType}"
                    }
                }
            }
        }

        stage('Check Configuration Changes') {
            steps {
                script {
                    def filesToCheck = []

                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        filesToCheck.add(
                            "environment/${params.ENVIRONMENT}.json"
                        )
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        filesToCheck.add(
                            "node/${params.ENVIRONMENT}.json"
                        )
                    }

                    def diffStatus = sh(
                        script: "git diff --quiet -- ${filesToCheck.join(' ')}",
                        returnStatus: true
                    )

                    if (diffStatus == 0) {
                        env.CONFIG_CHANGES = 'false'
                        echo 'No configuration changes detected. Git stages will be skipped.'
                    } else if (diffStatus == 1) {
                        env.CONFIG_CHANGES = 'true'
                        echo 'Configuration changes detected. Git stages will run.'
                    } else {
                        error('Unable to check configuration changes.')
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool(
                        name: 'SonarQube-Scanner',
                        type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                    )

                    withSonarQubeEnv('SonarQube-Server') {
                        withEnv(["PATH+SONAR=${scannerHome}/bin"]) {
                            sh '''
                                set -eu
                                sonar-scanner \
                                  -Dsonar.projectKey=flipkart-config-automation \
                                  -Dsonar.projectName=Flipkart-Config-Automation \
                                  -Dsonar.sources=environment,node \
                                  -Dsonar.sourceEncoding=UTF-8
                            '''
                        }
                    }
                }
            }
        }

        stage('Maven Compilation') {
            when {
                expression { fileExists('pom.xml') }
            }
            steps {
                sh 'mvn -B clean verify'
            }
        }

        stage('Create ZIP Artifact') {
            steps {
                script {
                    def files = []

                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        files.add("environment/${params.ENVIRONMENT}.json")
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        files.add("node/${params.ENVIRONMENT}.json")
                    }

                    env.ARTIFACT_NAME = "flipkart-config-${params.ENVIRONMENT}-${env.BUILD_NUMBER}.zip"
                    env.PACKAGE_FILES = files.join(',')

                    sh '''
                        set -eu
                        python3 - <<'PY'
import os
import zipfile

artifact = os.environ["ARTIFACT_NAME"]
files = os.environ["PACKAGE_FILES"].split(",")

with zipfile.ZipFile(artifact, "w", zipfile.ZIP_DEFLATED) as z:
    for path in files:
        if not os.path.isfile(path):
            raise FileNotFoundError(path)
        z.write(path)

print("Created:", artifact)
print("Contents:", files)
PY
                    '''

                    archiveArtifacts artifacts: env.ARTIFACT_NAME, fingerprint: true
                }
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x
                        curl --fail --show-error --silent \
                          --user "$NEXUS_USER:$NEXUS_PASSWORD" \
                          --upload-file "$ARTIFACT_NAME" \
                          "$NEXUS_REPOSITORY_URL/$ARTIFACT_NAME"
                        echo "Artifact uploaded to Nexus"
                    '''
                }
            }
        }

        stage('Create Git Feature Branch') {
            when {
                expression { env.CONFIG_CHANGES == 'true' }
            }
            steps {
                script {
                    env.FEATURE_BRANCH = "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"

                    sh '''
                        set -eu
                        git config user.name "Asif Shaik"
                        git config user.email "asifshaik6558@gmail.com"
                        git switch -c "$FEATURE_BRANCH"

                        if [ "$CONFIG_TYPE" = "environment" ]; then
                            git add -- "environment/$ENVIRONMENT.json"
                        elif [ "$CONFIG_TYPE" = "node" ]; then
                            git add -- "node/$ENVIRONMENT.json"
                        else
                            git add -- "environment/$ENVIRONMENT.json" "node/$ENVIRONMENT.json"
                        fi

                        git diff --cached --check

                        if git diff --cached --quiet; then
                            echo "Expected configuration changes were not staged."
                            exit 1
                        fi

                        git diff --cached --stat
                        git commit -m "Update $ENVIRONMENT configuration"
                    '''
                }
            }
        }

        stage('Push Feature Branch') {
            when {
                expression { env.CONFIG_CHANGES == 'true' }
            }
            steps {
                withCredentials([
                    gitUsernamePassword(credentialsId: 'git', gitToolName: 'Default')
                ]) {
                    sh '''
                        set -eu
                        git push -u origin "$FEATURE_BRANCH"
                    '''
                }
            }
        }

        stage('Create GitHub Pull Request') {
            when {
                expression { env.CONFIG_CHANGES == 'true' }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'git',
                        usernameVariable: 'GITHUB_USER',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        python3 - <<'PY'
import json
import os
import urllib.parse
import urllib.request

repo = os.environ["GITHUB_REPO"]
branch = os.environ["FEATURE_BRANCH"]
base = os.environ["BASE_BRANCH"]
token = os.environ["GITHUB_TOKEN"]
owner = repo.split("/")[0]

headers = {
    "Authorization": f"Bearer {token}",
    "Accept": "application/vnd.github+json",
    "X-GitHub-Api-Version": "2022-11-28"
}

query = urllib.parse.urlencode({
    "state": "open",
    "head": f"{owner}:{branch}",
    "base": base
})

request = urllib.request.Request(
    f"https://api.github.com/repos/{repo}/pulls?{query}",
    headers=headers
)

with urllib.request.urlopen(request, timeout=30) as response:
    pulls = json.load(response)

if pulls:
    print("Existing pull request:", pulls[0]["html_url"])
else:
    payload = {
        "title": f"Configuration update: {branch}",
        "head": branch,
        "base": base,
        "body": "Automated configuration update created by Jenkins."
    }

    request = urllib.request.Request(
        f"https://api.github.com/repos/{repo}/pulls",
        data=json.dumps(payload).encode("utf-8"),
        headers={**headers, "Content-Type": "application/json"},
        method="POST"
    )

    with urllib.request.urlopen(request, timeout=30) as response:
        result = json.load(response)
        print("Pull request created:", result["html_url"])
PY
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Configuration pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Console Output for the failed stage.'
        }

        always {
            echo "Final build result: ${currentBuild.currentResult}"
        }
    }
}
