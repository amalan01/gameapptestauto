pipeline {
    agent any

    environment {
        // ----------------------------------------------------------------
        // Credentials and Image config
        // ----------------------------------------------------------------
        DOCKERHUB_CREDENTIALS = 'cybr-3120'
        IMAGE_NAME = 'amalan06/amalangametest123'

        // ----------------------------------------------------------------
        // Trivy image scanning config
        // ----------------------------------------------------------------
        TRIVY_SEVERITY = "HIGH,CRITICAL"

        // ----------------------------------------------------------------
        // OWASP ZAP DAST config
        // ----------------------------------------------------------------
        TARGET_URL = "http://172.238.162.6/"
        REPORT_HTML = "zap_report.html"
        REPORT_JSON = "zap_report.json"
        ZAP_IMAGE = "ghcr.io/zaproxy/zaproxy:stable"
        REPORT_DIR = "${env.WORKSPACE}/zap_reports"
    }

    stages {

        /* ============================================================
           1) CLONE SOURCE CODE
        ============================================================ */
        stage('Cloning Git') {
            steps {
                checkout scm
            }
        }

        /* ============================================================
           2) SAST — SNYK (NON-BLOCKING)
        ============================================================ */
        stage('SAST-TEST') {
            steps {
                script {
                    echo "Running Snyk scan (will not stop pipeline)..."
                    try {
                        snykSecurity(
                            snykInstallation: 'Snyk-installations',
                            snykTokenId: 'Snyk-API-token',
                            severity: 'critical'
                        )
                    } catch (err) {
                        echo "Snyk found vulnerabilities but pipeline continues"
                        echo "Snyk Error: ${err}"
                        currentBuild.result = "UNSTABLE"   // does NOT stop pipeline
                    }
                }
            }
        }


        /* ============================================================
           3) SAST — SONARQUBE
        ============================================================ */
        stage('SonarQube Analysis') {
            agent { label 'CYBR3120-01-app-server' }
            steps {
                script {
                    def scannerHome = tool 'SonarQube-Scanner'
                    withSonarQubeEnv('SonarQube-installations') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=gameapp \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }


        /* ============================================================
           4) BUILD & TAG IMAGE
        ============================================================ */
        stage('BUILD-AND-TAG') {
            agent { label 'CYBR3120-01-app-server' }
            steps {
                script {
                    echo "Building Docker image ${IMAGE_NAME}..."
                    app = docker.build("${IMAGE_NAME}")
                    app.tag("latest")
                }
            }
        }


        /* ============================================================
           5) PUSH IMAGE TO DOCKER HUB
        ============================================================ */
        stage('POST-TO-DOCKERHUB') {
            agent { label 'CYBR3120-01-app-server' }
            steps {
                script {
                    echo "Pushing image to DockerHub..."
                    docker.withRegistry('https://registry.hub.docker.com', "${DOCKERHUB_CREDENTIALS}") {
                        app.push("latest")
                    }
                }
            }
        }


        /* ============================================================
           6) IMAGE SCAN — TRIVY (NON-BLOCKING)
        ============================================================ */
        stage("SECURITY-IMAGE-SCANNER") {
            steps {
                script {
                    echo "Running Trivy vulnerability scan..."

                    // JSON output
                    sh """
                        docker run --rm -v \$(pwd):/workspace aquasec/trivy:latest image \
                        --exit-code 0 \
                        --format json \
                        --output /workspace/trivy-report.json \
                        --severity ${TRIVY_SEVERITY} \
                        ${IMAGE_NAME}
                    """

                    // HTML output
                    sh """
                        docker run --rm -v \$(pwd):/workspace aquasec/trivy:latest image \
                        --exit-code 0 \
                        --format template \
                        --template "@/contrib/html.tpl" \
                        --output /workspace/trivy-report.html \
                        ${IMAGE_NAME}
                    """

                    archiveArtifacts artifacts: "trivy-report.json,trivy-report.html"
                }
            }
        }

        /* ============================================================
           7) SUMMARIZE TRIVY (NO STOP ON CRITICAL CVE)
        ============================================================ */
        stage("Summarize Trivy Findings") {
            steps {
                script {
                    if (!fileExists("trivy-report.json")) {
                        echo "No Trivy report found."
                        return
                    }

                    def highCount = sh(
                        script: "grep -o '\"Severity\": \"HIGH\"' trivy-report.json | wc -l",
                        returnStdout: true
                    ).trim()

                    def criticalCount = sh(
                        script: "grep -o '\"Severity\": \"CRITICAL\"' trivy-report.json | wc -l",
                        returnStdout: true
                    ).trim()

                    echo "HIGH Vulns: ${highCount}"
                    echo "CRITICAL Vulns: ${criticalCount}"

                    if (criticalCount.toInteger() > 0) {
                        echo "Critical Trivy vulnerabilities found. Build marked UNSTABLE but continuing..."
                        currentBuild.result = "UNSTABLE"     // does NOT stop pipeline
                    }
                }
            }
        }


        /* ============================================================
           8) PULL IMAGE ON SERVER
        ============================================================ */
        stage('Pull-image-server') {
            steps {
                sh "docker pull ${IMAGE_NAME}:latest || true"
            }
        }


        /* ============================================================
           9) DAST — OWASP ZAP FULL SCAN
        ============================================================ */
        stage('DAST') {
            steps {
                script {
                    echo "Running OWASP ZAP DAST..."

                    sh "mkdir -p ${REPORT_DIR}"

                    sh """
                        docker run --rm --user root --network host \
                        -v ${REPORT_DIR}:/zap/wrk \
                        -t ${ZAP_IMAGE} zap-baseline.py \
                        -t ${TARGET_URL} \
                        -r ${REPORT_HTML} -J ${REPORT_JSON}
                    """

                    archiveArtifacts artifacts: "zap_reports/*"
                }
            }
        }


        /* ============================================================
           10) DEPLOYMENT
        ============================================================ */
        stage('DEPLOYMENT') {
            agent { label 'CYBR3120-01-app-server' }
            steps {
                script {
                    echo "Starting docker-compose deployment..."

                    dir("${WORKSPACE}") {
                        sh """
                            docker-compose down || true
                            docker-compose up -d
                            docker ps
                        """
                    }
                }
                echo 'Deployment completed successfully!'
            }
        }
    }


    /* ============================================================
       PUBLISH HTML SECURITY REPORTS
    ============================================================ */
    post {
        always {
            publishHTML(target: [
                reportName: 'Trivy Image Security Report',
                reportDir: '.',
                reportFiles: 'trivy-report.html',
                alwaysLinkToLastBuild: true
            ])

            publishHTML(target: [
                reportName: 'OWASP ZAP DAST Report',
                reportDir: 'zap_reports',
                reportFiles: 'zap_report.html',
                alwaysLinkToLastBuild: true
            ])
        }
    }
}
