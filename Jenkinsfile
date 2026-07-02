pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "${JOB_NAME.toLowerCase().replaceAll('[^a-z0-9-]', '-')}"
        DOCKER_TAG   = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            when { expression { return fileExists('Dockerfile') } }
            steps {
                script {
                    def dockerAvailable = sh(script: 'which docker 2>/dev/null || test -x /usr/bin/docker', returnStatus: true) == 0
                    if (dockerAvailable) {
                        def daemonOk = sh(script: 'docker info > /dev/null 2>&1', returnStatus: true) == 0
                        if (daemonOk) {
                            retry(2) {
                                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                            }
                            sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
                        } else {
                            echo 'Docker daemon not reachable — run: docker exec jenkins chmod 666 /var/run/docker.sock'
                        }
                    } else {
                        echo 'Docker not available — install Docker in the Jenkins image or mount the socket'
                    }
                }
            }
        }

        stage('Trivy Scan') {
            when { expression { return fileExists('Dockerfile') } }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    script {
                        withEnv(["PATH+DEVPILOT=${env.HOME}/devpilot-tools/bin"]) {
                            def trivyOk = sh(script: 'which trivy 2>/dev/null', returnStatus: true) == 0
                            if (trivyOk) {
                                sh "trivy image --exit-code 0 --severity HIGH,CRITICAL --format table ${DOCKER_IMAGE}:${DOCKER_TAG} | tee trivy-report.txt"
                                archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
                            } else {
                                echo 'Trivy not available — skipping scan'
                            }
                        }
                    }
                }
            }
        }

        stage('Push to Registry') {
            when { expression { return fileExists('Dockerfile') } }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'devpilot-registry-1782053771939', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                            sh '''
                                BRANCH_TAG=$(echo ${GIT_BRANCH:-${BRANCH_NAME:-main}} | sed 's|origin/||' | tr '/' '-' | tr '[:upper:]' '[:lower:]')
                                echo $REG_PASS | docker login -u $REG_USER --password-stdin
                                docker tag $DOCKER_IMAGE:$DOCKER_TAG pav30/it-management-system-frontend:$DOCKER_TAG-$BRANCH_TAG
                                docker push pav30/it-management-system-frontend:$DOCKER_TAG-$BRANCH_TAG
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy to VM') {
            when { expression { return fileExists('Dockerfile') } }
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        withCredentials([sshUserPrivateKey(credentialsId: 'devpilot-deploy-Devlauch-IT-Management-System-main', keyFileVariable: 'SSH_KEY'), usernamePassword(credentialsId: 'devpilot-registry-1782053771939', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                            sh '''
                                BRANCH_TAG=$(echo ${GIT_BRANCH:-${BRANCH_NAME:-main}} | sed 's|origin/||' | tr '/' '-' | tr '[:upper:]' '[:lower:]')
                                REG_PASS_B64=$(echo -n "$REG_PASS" | base64 -w0)
                                ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=15 ubuntu@32.192.236.27 "echo $REG_PASS_B64 | base64 -d | docker login -u $REG_USER --password-stdin"
                                ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=15 ubuntu@32.192.236.27 "mkdir -p ~/devpilot-app && echo 'Tk9ERV9FTlY9ZGV2ZWxvcG1lbnQKTkVYVF9QVUJMSUNfU0lURV9VUkw9aHR0cDovL2l0LW1hbmFnZW1lbnQtc3lzdGVtLmRldmxhdWNoLmNvbQpORVhUX1BVQkxJQ19BUElfQkFTRV9VUkw9aHR0cDovL2l0LW1hbmFnZW1lbnQtc3lzdGVtLmRldmxhdWNoLmNvbS9hcGkKTU9OR09EQl9VUkk9bW9uZ29kYjovL3Jvb3Q6cm9vdEBtb25nb2RiOjI3MDE3L0lUX01hbmFnZW1lbnRfU3lzdGVtP2F1dGhTb3VyY2U9YWRtaW4KREJfTkFNRT1JVF9NYW5hZ2VtZW50X1N5c3RlbQpTQUxUX1JPVU5EUz0xMApKV1RfU0VDUkVUPXlvdXJfc2VjcmV0CkpXVF9FWFBJUkVTX0lOPTEwaApORVhUX1BVQkxJQ19QQUdFX1NJWkU9NTA=' | base64 -d > ~/devpilot-app/frontend.env"
                                ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=15 ubuntu@32.192.236.27 "pip3 install pyyaml -q 2>/dev/null || true; echo \"aW1wb3J0IHlhbWwsIHN5cywgb3MsIGZjbnRsCnBhdGggPSBvcy5wYXRoLmV4cGFuZHVzZXIoJ34vZGV2cGlsb3QtYXBwL2RvY2tlci1jb21wb3NlLnltbCcpCnRhZyA9IHN5cy5hcmd2WzFdCm9zLm1ha2VkaXJzKG9zLnBhdGguZXhwYW5kdXNlcignfi9kZXZwaWxvdC1hcHAnKSwgZXhpc3Rfb2s9VHJ1ZSkKbGYgPSBvcGVuKG9zLnBhdGguZXhwYW5kdXNlcignfi9kZXZwaWxvdC1hcHAvLmRldnBpbG90LmxvY2snKSwgJ3cnKQpmY250bC5mbG9jayhsZiwgZmNudGwuTE9DS19FWCkKdHJ5OgogdHJ5OgogIHdpdGggb3BlbihwYXRoKSBhcyBmOiBkYXRhID0geWFtbC5zYWZlX2xvYWQoZikgb3Ige30KIGV4Y2VwdCBFeGNlcHRpb246CiAgZGF0YSA9IHt9CiBpZiBub3QgaXNpbnN0YW5jZShkYXRhLmdldCgnc2VydmljZXMnKSwgZGljdCk6IGRhdGFbJ3NlcnZpY2VzJ10gPSB7fQogZXhpc3RpbmcgPSBkYXRhWydzZXJ2aWNlcyddLmdldCgnZnJvbnRlbmQnKQogcHJpbnQoJ1tkZXZwaWxvdF0gc2VydmljZT1mcm9udGVuZCBleGlzdGluZz0nICsgc3RyKGV4aXN0aW5nIGlzIG5vdCBOb25lKSkKIGlmIGV4aXN0aW5nOgogIHByaW50KCdbZGV2cGlsb3RdIG9sZCBpbWFnZT0nICsgc3RyKGV4aXN0aW5nLmdldCgnaW1hZ2UnKSkgKyAnIG9sZCBwb3J0cz0nICsgc3RyKGV4aXN0aW5nLmdldCgncG9ydHMnKSkpCiAgZXhpc3RpbmdbJ2ltYWdlJ10gPSAncGF2MzAvaXQtbWFuYWdlbWVudC1zeXN0ZW0tZnJvbnRlbmQ6JyArIHRhZwogIGV4aXN0aW5nWydjb250YWluZXJfbmFtZSddID0gJ2Zyb250ZW5kJwogIGV4aXN0aW5nWydwb3J0cyddID0gWyc3OTk6ODAnXQogIHByaW50KCdbZGV2cGlsb3RdIG5ldyBpbWFnZT0nICsgZXhpc3RpbmdbJ2ltYWdlJ10gKyAnIG5ldyBwb3J0cz0nICsgc3RyKGV4aXN0aW5nLmdldCgncG9ydHMnKSkpCiAgZGF0YVsnc2VydmljZXMnXVsnZnJvbnRlbmQnXSA9IGV4aXN0aW5nCiBlbHNlOgogIHByaW50KCdbZGV2cGlsb3RdIGNyZWF0aW5nIG5ldyBzZXJ2aWNlIGJsb2NrJykKICBzdmMgPSB7J2ltYWdlJzogJ3BhdjMwL2l0LW1hbmFnZW1lbnQtc3lzdGVtLWZyb250ZW5kOicgKyB0YWcsICdjb250YWluZXJfbmFtZSc6ICdmcm9udGVuZCcsICdyZXN0YXJ0JzogJ3VubGVzcy1zdG9wcGVkJ30KICBzdmNbJ3BvcnRzJ10gPSBbJzc5OTo4MCddCiAgc3ZjWydlbnZfZmlsZSddID0gWyd+L2RldnBpbG90LWFwcC9mcm9udGVuZC5lbnYnXQogIHByaW50KCdbZGV2cGlsb3RdIG5ldyBzZXJ2aWNlIHBvcnRzPScgKyBzdHIoc3ZjLmdldCgncG9ydHMnKSkpCiAgZGF0YVsnc2VydmljZXMnXVsnZnJvbnRlbmQnXSA9IHN2YwogaWYgbm90IGRhdGFbJ3NlcnZpY2VzJ10uZ2V0KCdtb25nb2RiJyk6CiAgZGF0YVsnc2VydmljZXMnXVsnbW9uZ29kYiddID0geydpbWFnZSc6ICdtb25nbzo3LjAnLCAnZW52aXJvbm1lbnQnOiB7J01PTkdPX0lOSVREQl9ST09UX1VTRVJOQU1FJzogJ3Jvb3QnLCAnTU9OR09fSU5JVERCX1JPT1RfUEFTU1dPUkQnOiAncm9vdCcsICdNT05HT19JTklUREJfREFUQUJBU0UnOiAnSVRfTWFuYWdlbWVudF9TeXN0ZW0nfSwgJ3Jlc3RhcnQnOiAndW5sZXNzLXN0b3BwZWQnLCAnaGVhbHRoY2hlY2snOiB7J3Rlc3QnOiBbJ0NNRC1TSEVMTCcsICdtb25nb3NoIC0tcXVpZXQgLS1ldmFsICJkYi5hZG1pbkNvbW1hbmQoe3Bpbmc6MX0pIiAtdSByb290IC1wIHJvb3QgLS1hdXRoZW50aWNhdGlvbkRhdGFiYXNlIGFkbWluIHx8IGV4aXQgMSddLCAnaW50ZXJ2YWwnOiAnNXMnLCAndGltZW91dCc6ICcxMHMnLCAncmV0cmllcyc6IDEwLCAnc3RhcnRfcGVyaW9kJzogJzQwcyd9LCAndm9sdW1lcyc6IFsnbW9uZ29kYXRhOi9kYXRhL2RiJ119CiBpZiBub3QgaXNpbnN0YW5jZShkYXRhLmdldCgndm9sdW1lcycpLCBkaWN0KTogZGF0YVsndm9sdW1lcyddID0ge30KIGRhdGFbJ3ZvbHVtZXMnXVsnbW9uZ29kYXRhJ10gPSB7fQogY3VyID0gZGF0YVsnc2VydmljZXMnXVsnZnJvbnRlbmQnXQogaWYgbm90IGlzaW5zdGFuY2UoY3VyLmdldCgnZGVwZW5kc19vbicpLCBkaWN0KTogY3VyWydkZXBlbmRzX29uJ10gPSB7fQogY3VyWydkZXBlbmRzX29uJ11bJ21vbmdvZGInXSA9IHsnY29uZGl0aW9uJzogJ3NlcnZpY2VfaGVhbHRoeSd9CiBpZiBub3QgaXNpbnN0YW5jZShjdXIuZ2V0KCdlbnZpcm9ubWVudCcpLCBkaWN0KTogY3VyWydlbnZpcm9ubWVudCddID0ge30KIGN1clsnZW52aXJvbm1lbnQnXVsnTU9OR09EQl9VUkwnXSA9ICdtb25nb2RiOi8vcm9vdDpyb290QG1vbmdvZGI6MjcwMTcvSVRfTWFuYWdlbWVudF9TeXN0ZW0/YXV0aFNvdXJjZT1hZG1pbicKIGRhdGFbJ3NlcnZpY2VzJ11bJ2Zyb250ZW5kJ10gPSBjdXIKIHdpdGggb3BlbihwYXRoLCAndycpIGFzIGY6IHlhbWwuZHVtcChkYXRhLCBmLCBkZWZhdWx0X2Zsb3dfc3R5bGU9RmFsc2UpCiBwcmludCgnW2RldnBpbG90XSB3cml0dGVuIHRvICcgKyBwYXRoKQogcHJpbnQoJ2Zyb250ZW5kIC0+IHBhdjMwL2l0LW1hbmFnZW1lbnQtc3lzdGVtLWZyb250ZW5kOicgKyB0YWcpCmZpbmFsbHk6CiBmY250bC5mbG9jayhsZiwgZmNudGwuTE9DS19VTikKIGxmLmNsb3NlKCk=\" | base64 -d > /tmp/devpilot_frontend.py"
                                PREV_TAG=$(ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=15 ubuntu@32.192.236.27 "grep 'image: pav30/it-management-system-frontend:' ~/devpilot-app/docker-compose.yml 2>/dev/null | awk '{print \$NF}' | head -1 || echo ''")
                                ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=30 ubuntu@32.192.236.27 "python3 /tmp/devpilot_frontend.py ${BUILD_NUMBER}-${BRANCH_TAG}"
                                COMPOSE_CMD=$(ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=15 ubuntu@32.192.236.27 "docker compose version >/dev/null 2>&1 && echo 'docker compose' || echo 'docker-compose'")
                                ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=60 ubuntu@32.192.236.27 "cd ~/devpilot-app && $COMPOSE_CMD pull frontend && $COMPOSE_CMD up -d" || {
                                    echo "Deploy failed — rolling back to $PREV_TAG"
                                    [ -n "$PREV_TAG" ] && ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no -o ConnectTimeout=15 ubuntu@32.192.236.27 "sed -i 's|image: pav30/it-management-system-frontend:.*|image: $PREV_TAG|' ~/devpilot-app/docker-compose.yml && cd ~/devpilot-app && $COMPOSE_CMD up -d" || true
                                    exit 1
                                }
                                echo "Deployed frontend to http://32.192.236.27"
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                def status = currentBuild.result ?: 'IN_PROGRESS'
                def promptText = "Analyze this Jenkins CI/CD build and give 2-3 actionable bullet points: what passed, what failed (if any), and one improvement.\nJob: ${env.JOB_NAME} Build#${env.BUILD_NUMBER} Branch: ${env.GIT_BRANCH ?: env.BRANCH_NAME ?: 'unknown'} Status: ${status}"
                def aiDone = false

                for (def credId : ['devpilot-anthropic-key', 'ANTHROPIC_API_KEY']) {
                    if (aiDone) break
                    try {
                        withCredentials([string(credentialsId: credId, variable: 'ANTHROPIC_KEY')]) {
                            writeFile file: '.ai-payload.json', text: groovy.json.JsonOutput.toJson([
                                model: 'claude-haiku-4-5-20251001',
                                max_tokens: 350,
                                messages: [[role: 'user', content: promptText]]
                            ])
                            def rc = sh returnStatus: true, script: '''
                                curl -sf -X POST https://api.anthropic.com/v1/messages \
                                  -H 'Content-Type: application/json' \
                                  -H "x-api-key: $ANTHROPIC_KEY" \
                                  -H 'anthropic-version: 2023-06-01' \
                                  --max-time 30 \
                                  -d @.ai-payload.json \
                                  -o .ai-response.json
                            '''
                            if (rc == 0) {
                                def resp = new groovy.json.JsonSlurper().parseText(readFile('.ai-response.json'))
                                echo "\n=== Claude AI Build Analysis ===\n${resp.content[0].text}\n================================"
                                writeFile file: 'ai-analysis.json', text: readFile('.ai-response.json')
                                archiveArtifacts artifacts: 'ai-analysis.json', allowEmptyArchive: true
                                aiDone = true
                            }
                        }
                    } catch (ignored) {}
                }

                for (def credId : ['devpilot-openai-key', 'OPENAI_API_KEY']) {
                    if (aiDone) break
                    try {
                        withCredentials([string(credentialsId: credId, variable: 'OPENAI_KEY')]) {
                            writeFile file: '.ai-payload.json', text: groovy.json.JsonOutput.toJson([
                                model: 'gpt-4o-mini',
                                max_tokens: 350,
                                messages: [[role: 'user', content: promptText]]
                            ])
                            def rc = sh returnStatus: true, script: '''
                                curl -sf -X POST https://api.openai.com/v1/chat/completions \
                                  -H 'Content-Type: application/json' \
                                  -H "Authorization: Bearer $OPENAI_KEY" \
                                  --max-time 30 \
                                  -d @.ai-payload.json \
                                  -o .ai-response.json
                            '''
                            if (rc == 0) {
                                def resp = new groovy.json.JsonSlurper().parseText(readFile('.ai-response.json'))
                                echo "\n=== ChatGPT Build Analysis ===\n${resp.choices[0].message.content}\n==============================="
                                writeFile file: 'ai-analysis.json', text: readFile('.ai-response.json')
                                archiveArtifacts artifacts: 'ai-analysis.json', allowEmptyArchive: true
                                aiDone = true
                            }
                        }
                    } catch (ignored) {}
                }

                if (!aiDone) {
                    echo 'AI analysis skipped — configure an API key in DevPilot Settings (Claude or ChatGPT)'
                }
            }
        }
        success { echo 'Pipeline succeeded!' }
        failure  { echo 'Pipeline failed!' }
    }
}