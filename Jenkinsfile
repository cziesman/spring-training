library identifier: "pipeline-library@v1.5",
        retriever: modernSCM(
                [
                        $class: "GitSCMSource",
                        remote: "https://github.com/redhat-cop/pipeline-library.git"
                ]
        )

openshift.withCluster() {
    env.NAMESPACE = openshift.project()
    env.POM_FILE = env.BUILD_CONTEXT_DIR ? "${env.BUILD_CONTEXT_DIR}/pom.xml" : "pom.xml"
    env.APP_NAME = "${JOB_NAME}".replaceAll(/-build.*/, '')
    env.BUILD = "${env.NAMESPACE}"
    env.BUILD_CONFIG = "${APP_NAME}-binary"
    env.APPLICATION_SOURCE_REPO = "https://github.com/cziesman/spring-training.git"
    env.APPLICATION_SOURCE_REF = "master"
    env.DEV_TAG = "dev"
    env.DEV = "${APP_NAME}-dev"
    env.STAGE_TAG = "stage"
    env.STAGE = "${APP_NAME}-stage"
    env.BUILD_OUTPUT_DIR = "target"
    env.VERSION = ""
    //env.SRC_REGISTRY = "${SRC_REGISTRY}"
    //env.DEST_REGISTRY = "${DEST_REGISTRY}"
    //env.DEST_CREDENTIALS = "${DEST_CREDENTIALS}"
    //env.DEST_PROJECT = "${DEST_PROJECT}"
    //env.SONAR_HOST_URL = "${SONAR_HOST_URL}"
    //env.SONAR_LOGIN = "${SONAR_LOGIN}"
    //env.SONAR_PASSWORD = "${SONAR_PASSWORD}"
    //env.SETTINGS_FILE_ID = "nexus_credentials_file"
    echo "Starting Pipeline for ${APP_NAME}..."
}

pipeline {
    agent {
        label 'maven'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git url: "${env.APPLICATION_SOURCE_REPO}", branch: "${env.APPLICATION_SOURCE_REF}"
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    env.VERSION = pom.version
                }
            }
        }

        stage('Build, Unit Test, Coverage Checks') {
            steps {
                echo "Building, running tests, and running coverage checks"
                script {
                    try {
                        sh "mvn clean install -Ddependency-check.skip=true"

                        // publish unit test report
                        publishHTML(target: [
                                reportDir            : "${env.BUILD_OUTPUT_DIR}/site/jacoco",
                                reportFiles          : 'index.html',
                                reportName           : 'Jacoco Unit Test Report',
                                keepAll              : true,
                                alwaysLinkToLastBuild: false,
                                allowMissing         : true
                        ])
                    } catch (err) {
                        echo err.getMessage()
                        throw err
                    }
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
                            openshift.newBuild("--name=${env.APP_NAME}", "--image-stream=openjdk-11-rhel7:1.1", "--binary=true")
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
                    try {
                        openshift.withCluster() {
                            openshift.withProject("${env.DEV}") {
                                openshift.loglevel(5)
                                def buildSelector = openshift.selector("bc", "${env.BUILD_CONFIG}")
                                        .startBuild("--from-dir=oc-build/deployments", "--namespace=${env.DEV}")
                                buildSelector.logs('-f')
                            }
                        }
                    } catch (err) {
                        echo "******************** Image creation failed *******************"
                        echo err.getMessage()
                        throw err
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
                            openshift.tag("${env.DEV}/${env.APP_NAME}:latest", "${env.DEV}/${env.APP_NAME}:${env.DEV_TAG}")
                        }
                    }
                }
            }
        }

        stage('Create service and route in DEV') {
            // this is not how one would normally expose a service and a route,
            // but it's a simpler approach for the purposes of this training exercise.
            steps {
                script {
                    try {
                        openshift.withCluster() {
                            openshift.withProject("${env.DEV}") {
                                def app = openshift.selector("dc", "${env.APP_NAME}")
                                if (!openshift.selector("svc", "${env.APP_NAME}").exists()) {
                                    echo "Creating service"
                                    app.narrow("svc").expose();
                                }
                                if (!openshift.selector("routes", "${env.APP_NAME}").exists()) {
                                    echo "Creating route"
                                    openshift.raw("expose svc/${env.APP_NAME}");
                                }
                            }
                        }
                    } catch (err) {
                        echo "******************** Service or route expose failed *******************"
                        echo err.getMessage()
                        throw err
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

                            def healthResponse = 1
                            timeout(time: 5, unit: 'MINUTES') {
                                waitUntil {
                                    def health = httpRequest(url: "http://${env.DEV_ROUTE}/health", validResponseCodes: "200:600")
                                    if (health.status == 200 && health.content.contains("{\"status\":\"UP\"}")) {
                                        healthResponse = 0
                                    }
                                    return (healthResponse == 0)
                                }
                            }
                            if (healthResponse != 0) {
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

                            def smokeResponse = 1
                            timeout(time: 5, unit: 'MINUTES') {
                                waitUntil {
                                    def smoke = httpRequest(url: "http://${env.DEV_ROUTE}", validResponseCodes: "200:600")
                                    if (smoke.status == 200 && smoke.content.contains("\"Hello, World!\"")) {
                                        smokeResponse = 0
                                    }
                                    return (smokeResponse == 0)
                                }

                                if (smokeResponse != 0) {
                                    error("Running smoke tests failed")
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('OWASP Vulnerability Checks') {
            steps {
                echo "Running OWASP vulnerability checks"
                script {
                    try {
                        sh "mvn dependency-check:check"

                        // publish dependency report
                        publishHTML(target: [
                                reportDir            : "${env.BUILD_OUTPUT_DIR}",
                                reportFiles          : 'dependency-check-report.html',
                                reportName           : 'OWASP Dependency Vulnerability Report',
                                keepAll              : true,
                                alwaysLinkToLastBuild: false,
                                allowMissing         : true
                        ])
                    } catch (err) {
                        echo err.getMessage()
                        throw err
                    }
                }
            }
        }

    }
}
