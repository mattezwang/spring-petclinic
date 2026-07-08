pipeline {
    agent any

    parameters {
        string(name: 'PROD_URL', defaultValue: 'http://192.168.64.10:8080',
               description: 'URL of spring-petclinic on the production VM, used as the ZAP scan target. Update after `multipass launch` gives you the VM IP.')
    }

    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
        ANSIBLE_DIR       = '/home/jenkins/ansible'
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

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Test') {
            steps {
                sh './mvnw -B clean verify'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "./mvnw -B sonar:sonar -Dsonar.projectKey=${SONAR_PROJECT_KEY}"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                sh './mvnw -B package -DskipTests'
                stash name: 'jar', includes: 'target/*.jar'
            }
        }

        stage('Deploy to Production VM (Ansible)') {
            steps {
                unstash 'jar'
                sh '''
                    JAR_FILE=$(ls target/*.jar | head -n1)
                    ansible-playbook -i ${ANSIBLE_DIR}/inventory.ini ${ANSIBLE_DIR}/deploy.yml \
                        --extra-vars "jar_src=${WORKSPACE}/${JAR_FILE}"
                '''
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                sh '''
                    mkdir -p zap-report
                    docker run --rm --network devsecops-net \
                        -v ${WORKSPACE}/zap-report:/zap/wrk/:rw \
                        zaproxy/zap-stable zap-baseline.py \
                        -t ${PROD_URL} \
                        -r zap-report.html \
                        -I || true
                '''
                // zap-baseline.py exits non-zero when it finds WARN/FAIL alerts,
                // which is expected during a normal scan, not a pipeline failure —
                // the report itself is what we publish and act on.
            }
        }

        stage('Publish Reports') {
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
