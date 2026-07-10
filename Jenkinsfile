pipeline {
    agent any

    parameters {
        string(name: 'PROD_URL', defaultValue: 'http://192.168.1.179:8080')
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
                    # jenkins uses the host docker daemon, so a bind mount of
                    # $WORKSPACE doesn't resolve correctly - use a named
                    # volume instead and copy the report out after
                    VOL="zap-wrk-${BUILD_NUMBER}"
                    docker volume create "$VOL" > /dev/null
                    docker run --rm -v "$VOL":/zap/wrk alpine chown -R 1000:1000 /zap/wrk

                    docker run --rm --network devsecops-net \
                        -v "$VOL":/zap/wrk/:rw \
                        zaproxy/zap-stable zap-baseline.py \
                        -t ${PROD_URL} \
                        -r zap-report.html \
                        -I || true

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
