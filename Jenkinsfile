pipeline {
    agent any

    environment {
        // Credentials & Configurations
        DOCKERHUB_CREDENTIALS = 'cybr-3120'
        IMAGE_NAME = 'amalan06/amalangametest123'

        // TRIVY
        TRIVY_SEVERITY = "HIGH,CRITICAL"

        // ZAP
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
           2) SAST - SNYK SCAN
        ============================================================ */
        stage('SAST-TEST') {
            steps {
                script {
                    snykSecurity(
                        snykInstallation: 'Snyk-installations',
                        snykTokenId: 'Snyk-API-token',
                        severity: 'critical'
                    )
                }
            }
        }

        /* ============================================================
           3) SAST - SONARQUBE
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
           4) BUILD & TAG DOCKER IMAGE
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
                    echo "Pushing image ${IMAGE_NAME}:latest to Docker Hub..."
                    docker.withRegistry('https://registry.hub.docker.com', "${DOCKERHUB_CREDENTIALS}") {
                        app.push("latest")
                    }
                }
            }
        }


        /* ============================================================
           6) IMAGE SECURITY SCAN — TRIVY
        ============================================================ */
        stage("SECURITY-IMAGE-SCANNER") {
            steps {
                echo "Running Trivy Container Security Scan..."

                // JSON report
                sh """
                    docker run --rm -v \$(pwd):/workspace aquasec/trivy:latest image \
                    --exit-code 0 \
                    --format json \
                    --output /workspace/trivy-report.json \
                    --severity ${TRIVY_SEVERITY} \
                    ${IMAGE_NAME}
                """

                // HTML report
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

        /* ============================================================
           7) SUMMARIZE TRIVY FINDINGS (DO NOT STOP PIPELINE)
        ============================================================ */
        stage("Summarize Trivy Vulnerabilities") {
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
                        currentBuild.result = "UNSTABLE"
                        echo "Build marked UNSTABLE due to critical CVEs — continuing with DAST..."
                    }
                }
            }
        }


        /* ============================================================
           8) PULL IMAGE ON SERVER (OPTIONAL)
        ============================================================ */
        stage('Pull-image-server') {
            steps {
                sh "docker pull ${IMAGE_NAME}:latest || true"
            }
        }

        /* ============================================================
           9) DAST — OWASP ZAP BASELINE SCAN
        ============================================================ */
        stage('DAST') {
            steps {
                script {
                    echo "Running OWASP ZAP..."

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
           10 DEPLOYMENT
        ============================================================ */
        stage('DEPLOYMENT') {
            agent { label 'CYBR3120-01-app-server' }
            steps {
                echo 'Starting deployment using docker-compose...'
                script {
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
       11) PUBLISH SECURITY REPORTS
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
