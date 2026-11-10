pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timeout(time: 5, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        AWX_URL = 'http://host.docker.internal:8080'
    }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'AWX_JOB_ID', value: '$.awx_job_id'],
                [key: 'AWX_STATUS', value: '$.status']
            ],
            tokenCredentialId: 'awx-webhook-token',
            causeString: 'Avviata da AWX job $AWX_JOB_ID',
            printContributedVariables: false,
            printPostContent: false,
            regexpFilterText: '$AWX_STATUS',
            regexpFilterExpression: '^successful$'
        )
    }

    stages {
        stage('Recupera il sorgente da AWX') {
            steps {
                deleteDir()

                script {
                    if (!(env.AWX_JOB_ID ==~ /[1-9][0-9]*/)) {
                        error('ID AWX assente o non valido. Dopo la prima inizializzazione, avvia il job da AWX.')
                    }
                    currentBuild.description = "AWX job ${env.AWX_JOB_ID}"
                }

                withCredentials([
                    string(credentialsId: 'awx-api-token', variable: 'AWX_TOKEN')
                ]) {
                    sh '''
                        set +x
                        set -eu
                        curl --fail --silent --show-error \
                          --connect-timeout 10 --max-time 60 \
                          -H "Authorization: Bearer $AWX_TOKEN" \
                          "$AWX_URL/api/v2/jobs/$AWX_JOB_ID/" \
                          -o awx-job.json
                    '''
                }

                script {
                    def job = readJSON file: 'awx-job.json', returnPojo: true
                    if (job.id.toString() != env.AWX_JOB_ID ||
                        job.status != 'successful' ||
                        !(job.artifacts?.sorgente_java_base64 instanceof String) ||
                        !job.artifacts.sorgente_java_base64.trim()) {
                        error('Il job AWX non corrisponde, non è riuscito o non contiene il sorgente.')
                    }
                    writeFile file: 'sorgente.b64', text: job.artifacts.sorgente_java_base64
                }

                sh '''
                    set -eu
                    base64 --decode sorgente.b64 > Example.java
                    test -s Example.java
                    rm awx-job.json sorgente.b64
                '''
            }
        }

        stage('Crea artefatto sorgente') {
            steps {
                sh 'tar -czf source.tar.gz Example.java'
                archiveArtifacts artifacts: 'source.tar.gz', fingerprint: true
                sh 'rm Example.java'
            }
        }

        stage('Compila dall’artefatto') {
            steps {
                sh '''
                    set -eu
                    mkdir build
                    tar -xzf source.tar.gz -C build
                    javac -encoding UTF-8 -d build build/Example.java
                '''
                archiveArtifacts artifacts: 'build/Example.class', fingerprint: true
            }
        }
    }

    post {
        always {
            deleteDir()
        }
    }
}
