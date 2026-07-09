pipeline {
    agent any

    parameters {
        string(name: 'PROD_URL', defaultValue: 'http://192.168.68.59:8080',
               description: 'URL of spring-petclinic on the production VM, used as the ZAP scan target. Update after `multipass launch` gives you the VM IP.')
    }

    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
        ANSIBLE_DIR       = '/home/jenkins/ansible'
        // ansible-playbook runs from $WORKSPACE below, not from ANSIBLE_DIR,
        // so ansible.cfg (host_key_checking = False) wouldn't be picked up
        // by cwd-based discovery alone — point at it explicitly.
        ANSIBLE_CONFIG    = '/home/jenkins/ansible/ansible.cfg'
    }

    options {
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    triggers {
        // "Set up build triggers to poll Source Control Management (SCM)."
        pollSCM('H/2 * * * *')
    }

    stages {

        stage('checkout') {
            steps {
                checkout scm
            }
        }

        stage('build / unit Test') {
            steps {
                sh './mvnw -B clean verify'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('sonarqube') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "./mvnw -B sonar:sonar -Dsonar.projectKey=${SONAR_PROJECT_KEY}"
                }
            }
        }

        stage('quality') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('package') {
            steps {
                sh './mvnw -B package -DskipTests'
                stash name: 'jar', includes: 'target/*.jar'
            }
        }

        stage('deploy to prod VM (Ansible)') {
            steps {
                unstash 'jar'
                sh '''
                    JAR_FILE=$(ls target/*.jar | head -n1)
                    ansible-playbook -i ${ANSIBLE_DIR}/inventory.ini ${ANSIBLE_DIR}/deploy.yml \
                        --extra-vars "jar_src=${WORKSPACE}/${JAR_FILE}"
                '''
            }
        }

        stage('OWASP ZAP scan') {
            steps {
                sh '''
                    # Jenkins talks to the host Docker daemon over the mounted
                    # socket (Docker-outside-of-Docker), so a bind mount like
                    # -v ${WORKSPACE}/zap-report:/zap/wrk resolves against the
                    # daemon's own filesystem, not this container's — it looks
                    # like it works but the report silently goes nowhere. A
                    # named volume plus `docker run ... cat | >` sidesteps that
                    # because it streams through the Docker API instead of a
                    # host path. ZAP's container also runs as uid 1000, so the
                    # freshly-created (root-owned) volume needs a chown first.
                    VOL="zap-wrk-${BUILD_NUMBER}"
                    docker volume create "$VOL" > /dev/null
                    docker run --rm -v "$VOL":/zap/wrk alpine chown -R 1000:1000 /zap/wrk

                    docker run --rm --network devsecops-net \
                        -v "$VOL":/zap/wrk/:rw \
                        zaproxy/zap-stable zap-baseline.py \
                        -t ${PROD_URL} \
                        -r zap-report.html \
                        -I || true
                    # zap-baseline.py exits non-zero when it finds WARN/FAIL
                    # alerts, which is expected during a normal scan, not a
                    # pipeline failure — the report itself is what we publish.

                    mkdir -p zap-report
                    docker run --rm -v "$VOL":/zap/wrk/:ro alpine cat /zap/wrk/zap-report.html > zap-report/zap-report.html
                    docker volume rm "$VOL" > /dev/null
                '''
            }
        }

        stage('publish') {
            steps {
                publishHTML(target: [
                    reportDir: 'zap-report',
                    reportFiles: 'zap-report.html',
                    reportName: 'ZAP Security Report',
                    keepAll: true,
                    alwaysLinkToLastBuild: true
                ])
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
