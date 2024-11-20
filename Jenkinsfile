pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
    }
    stages {
        stage('Code checkout from GitHub') {
            steps {
                script {
                    cleanWs()
                    git credentialsId: 'github-token', url: 'https://github.com/jarzyna63/abcd-student', branch: 'main'
                }
            }
        }
        stage('Preparation stage') {
            steps {
                sh 'mkdir -p results/'
            }
        }
        stage('SCA scan') {
            steps {
                sh 'osv-scanner scan --lockfile package-lock.json --format json --output results/sca-osv-scanner.json'
            }
            post {
                always {
                    defectDojoPublisher(artifact: 'results/sca-osv-scanner.json', 
                        productName: 'Juice Shop', 
                        scanType: 'OSV Scan', 
                        engagementName: 'krzysztof.czartoryski@xtb.com')
                    echo 'SCA scan succeeded!'
                }
            }
        }
        stage('DAST by [ZAP]') {
            steps {
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 10
                '''
                sh '''
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v /home/k1/workspace/kursy/ABCDevSecOps/abcd-student/.zap:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                        "zap.sh -cmd -addonupdate; zap.sh -cmd -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta -autorun /zap/wrk/passive.yaml" \
                        || true
                '''
                 sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html
                        docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml
                '''
            }
            post {
               
                success {
                    echo 'I succeeded!'
                    echo "Archiving results..."
                    archiveArtifacts artifacts: 'results/**/*', fingerprint: true, allowEmptyArchive: true
                    echo "Sending reports to Defect Dojo..."
                    defectDojoPublisher(artifact: 'results/zap_xml_report.xml', 
                        productName: 'Juice Shop', 
                        scanType: 'ZAP Scan', 
                        engagementName: 'krzysztof.czartoryski@xtb.com')
                }
                 always {
                    sh '''
                        docker stop juice-shop zap 
                        docker rm zap
                    '''  
                }
            }
        }
    }
}
