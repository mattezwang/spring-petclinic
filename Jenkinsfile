pipeline {
    agent any

    parameters {
        string(name: 'PROD_URL', defaultValue: 'http://192.168.68.59:8080')
    }

    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
        ANSIBLE_DIR       = '/home/jenkins/ansible'
        ANSIBLE_CONFIG    = '/home/jenkins/ansible/ansible.cfg'
    }

    options {
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    triggers {
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
            }
        }

        stage('deploy to prod VM (Ansible)') {
            steps {
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
                    mkdir -p zap-report
                    chmod 777 zap-report

                    docker run --rm --network devsecops-net \
                        -v "${WORKSPACE}/zap-report":/zap/wrk/:rw \
                        zaproxy/zap-stable zap-baseline.py \
                        -t ${PROD_URL} \
                        -r zap-report.html \
                        -I || true
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
