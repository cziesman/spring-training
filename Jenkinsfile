// setup the environment variables
openshift.withCluster() {
    env.NAMESPACE = openshift.project()
    env.POM_FILE = env.BUILD_CONTEXT_DIR ? "${env.BUILD_CONTEXT_DIR}/pom.xml" : "pom.xml"
    env.APP_NAME = "${JOB_NAME}".replaceAll(/-build.*/, '')
    env.BUILD = "${env.NAMESPACE}"
    env.BUILD_CONFIG = "${APP_NAME}-binary"
    env.APPLICATION_SOURCE_REPO = "https://github.com/cziesman/spring-training.git"
    env.APPLICATION_SOURCE_BRANCH = "master"
    env.DEV_TAG = "dev"
    env.DEV = "${APP_NAME}-dev"
    env.STAGE_TAG = "stage"
    env.STAGE = "${APP_NAME}-stage"
    env.BUILD_OUTPUT_DIR = "target"
    env.VERSION = ""
    env.SRC_REGISTRY = "default-route-openshift-image-registry.apps.ocp-cz.do280.dev.nextcle.com"
    env.SERVER = "https://api.ocp-cz.do280.dev.nextcle.com:6443"
    env.SRC_NAMESPACE="ci-cd"
    env.SRC_TAG="latest"
    env.DEST_REGISTRY = "quay.io"
    env.DEST_PROJECT = "cziesman0"
    env.DOCKER_DIR = "${HOME}/.docker"
    env.DOCKER_CONFIG = "${env.DOCKER_DIR}/config.json"

    echo "Starting Pipeline for ${APP_NAME}..."
}

// extract the build parameter for controlling OWASP vulnerability scans
Boolean SKIP_OWASP = Boolean.valueOf(params.SKIP_OWASP)

// save the config.json file to $HOME/.docker/config.json
void writeConfig(String configFile, String auths) {

    def file = new File(configFile)

    file.write(auths)
}

// encode the username:password as Base64
String base64(String text) {

    return text.bytes.encodeBase64().toString()
}

// create the auth entry JSON string for a registry
String creds(String name, String username, String password) {

    String auth = (username + ':' + password).bytes.encodeBase64().toString()

    String retval =
    """"${name}": {
                "auth": "${auth}"
            }"""

    return retval
}

// create the contents of the config.json file
String auths(String... creds) {

    String list = ""

    creds.eachWithIndex { item, index ->
  	     String term = (index < creds.length-1 ) ? ",\n            " : ""
         list += item + term
    }

    String retval =
    """    {
        "auths": {
            ${list}
        }
    }"""

    return retval
}

pipeline {
    agent any

    stages {

        stage('Git Checkout') {
            steps {
                echo "Checking out from Git"
                git url: "${env.APPLICATION_SOURCE_REPO}", branch: "${env.APPLICATION_SOURCE_BRANCH}"
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    env.VERSION = pom.version
                }
            }
        }

        stage('Build, Test, Coverage Checks') {
            steps {
                echo "Building, running unit tests, and running coverage checks"
                script {
                    try {
                        // skip dependency checks as those are done in a later stage
                        sh "mvn clean install -Ddependency-check.skip=true"
                    } catch (err) {
                        echo err.getMessage()
                        throw err
                    }
                }
            }
            post {
                always {
                    // publish unit test report
                    publishHTML(target: [
                            reportDir            : "${env.BUILD_OUTPUT_DIR}/site/jacoco",
                            reportFiles          : 'index.html',
                            reportName           : 'Jacoco Unit Test Report',
                            keepAll              : true,
                            alwaysLinkToLastBuild: false,
                            allowMissing         : true
                    ])
                }
            }
        }

        stage('Create Image Builder') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject("${env.DEV}") {
                            return !openshift.selector("bc", "${env.BUILD_CONFIG}").exists();
                        }
                    }
                }
            }
            steps {
                echo "Creating image builder"
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.DEV}") {
                            try {
                                openshift.newBuild("--name=${env.APP_NAME}", "--image-stream=openjdk-11-rhel7:1.1", "--binary=true")
                            } catch (err) {
                                echo "******************** Image builder creation failed *******************"
                                echo err.getMessage()
                                throw err
                            }
                        }
                    }
                }
            }
        }

        stage('Create Image') {
            steps {
                echo "Creating image"
                sh "rm -rf oc-build && mkdir -p oc-build/deployments"
                sh "cp target/${env.APP_NAME}-${env.VERSION}.jar oc-build/deployments/${env.APP_NAME}.jar"
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.DEV}") {
                            try {
                                openshift.loglevel(2)
                                def buildSelector = openshift.selector("bc", "${env.BUILD_CONFIG}")
                                        .startBuild("--from-dir=oc-build/deployments", "--namespace=${env.DEV}")
                                buildSelector.logs('-f')

                            } catch (err) {
                                echo "******************** Image creation failed *******************"
                                echo err.getMessage()
                                throw err
                            }
                        }
                    }
                }
            }
        }

        stage('Tag DEV image') {
            steps {
                echo "Tagging DEV image"
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.DEV}") {
                            try {
                                openshift.tag("${env.DEV}/${env.APP_NAME}:latest", "${env.DEV}/${env.APP_NAME}:${env.DEV_TAG}")
                            } catch (err) {
                                echo "******************** Tagging DEV image failed *******************"
                                echo err.getMessage()
                                throw err
                            }
                        }
                    }
                }
            }
        }

        stage('Create service in DEV') {
            // this is not how one would normally expose a service and a route,
            // but it's a simpler approach for the purposes of this training exercise.
            steps {
                echo "Creating service and route in DEV"
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.DEV}") {
                            try {
                                def app = openshift.selector("dc", "${env.APP_NAME}")
                                if (!openshift.selector("svc", "${env.APP_NAME}").exists()) {
                                    echo "Creating service"
                                    app.narrow("svc").expose();
                                }
                                if (!openshift.selector("routes", "${env.APP_NAME}").exists()) {
                                    echo "Creating route"
                                    openshift.raw("expose svc/${env.APP_NAME}");
                                }
                            } catch (err) {
                                echo "******************** Service or route expose failed *******************"
                                echo err.getMessage()
                                throw err
                            }
                        }
                    }
                }
            }
        }

        stage("Verify deploy") {
            steps {
                echo "Verifying DEV deployment"
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.DEV}") {
                            env.DEV_ROUTE = openshift.selector("routes", "${env.APP_NAME}").narrow('route').object().spec.host

                            def healthResponse = false
                            timeout(time: 5, unit: 'MINUTES') {
                                waitUntil {
                                    def health = httpRequest(url: "http://${env.DEV_ROUTE}/health", validResponseCodes: "200:600")
                                    if (health.status == 200 && health.content.contains("{\"status\":\"UP\"}")) {
                                        healthResponse = true
                                    }
                                    return (healthResponse)
                                }
                            }
                            if (!healthResponse) {
                                error("Verify deploy failed")
                            }
                        }
                    }
                }
            }
        }

        stage("Run Smoke Tests") {
            steps {
                echo "Running smoke tests"
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.DEV}") {

                            def smokeResponse = false
                            timeout(time: 5, unit: 'MINUTES') {
                                waitUntil {
                                    def smoke = httpRequest(url: "http://${env.DEV_ROUTE}", validResponseCodes: "200:600")
                                    if (smoke.status == 200 && smoke.content.contains("\"Hello, World!\"")) {
                                        smokeResponse = true
                                    }
                                    return (smokeResponse)
                                }

                                if (!smokeResponse) {
                                    error("Running smoke tests failed")
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Run OWASP Checks') {
            when {
                expression {
                  return !SKIP_OWASP
                }
            }
            steps {
                script {
                    echo "Running OWASP vulnerability checks"
                    sh "mvn dependency-check:check"
                }
            }
            post {
                always {
                    // publish dependency report
                    publishHTML(target: [
                            reportDir            : "${env.BUILD_OUTPUT_DIR}",
                            reportFiles          : 'dependency-check-report.html',
                            reportName           : 'OWASP Dependency Vulnerability Report',
                            keepAll              : true,
                            alwaysLinkToLastBuild: false,
                            allowMissing         : true
                    ])
                }
            }
        }

        stage("Push Image to Quay") {
            steps {
                echo "Pushing Image to Quay"
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.DEV}") {
                            try {
                                // these credentials are defined as global credentials in jenkins.
                                withCredentials([usernamePassword(credentialsId: 'registry-secret', usernameVariable: 'srcUsername', passwordVariable: 'srcPassword')]) {
                                    withCredentials([usernamePassword(credentialsId: 'quay-secret', usernameVariable: 'destUsername', passwordVariable: 'destPassword')]) {

                                    // we have to build the config.json file manually because it's not
                                    // straightforward to use podman or docker to login to a registry
                                    // from Jenkins in an openshift container, and because for some unknown
                                    // reason skopeo cannot authenticate if we provide credentials as parameters.
                                    // this is an ugly, ugly hack but it gets the job done.

                                    // login to openshift so we can get a token for registry access
                                    sh "oc login -u ${srcUsername} -p ${srcPassword} --server=${env.SERVER}"
                                    def token = sh(script: "oc whoami -t", returnStdout: true).trim()

                                    // build the contents of the docker config.json file
                                    def srcCreds = creds(env.SRC_REGISTRY, srcUsername, token)
                                    def destCreds = creds(env.DEST_REGISTRY, destUsername, destPassword)
                                    def auths = auths(srcCreds, destCreds)

                                    // save the config.json file
                                    sh "mkdir -p ${env.DOCKER_DIR}"
                                    writeConfig(env.DOCKER_CONFIG, auths)

                                    // copy the image from the internal registry to Quay
                                    sh 'skopeo --debug copy --src-tls-verify=false --dest-tls-verify=false ' + \
                                       "docker://${env.SRC_REGISTRY}/${env.DEV}/${env.APP_NAME} " + \
                                       "docker://${env.DEST_REGISTRY}/${env.DEST_PROJECT}/${env.APP_NAME}:${env.VERSION}"
                                }
                              }
                            } catch (err) {
                                echo err.getMessage()
                                throw err
                              }
                        }
                    }
                }
            }
        }
    }
}
