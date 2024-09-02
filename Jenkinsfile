pipeline{
    agent any
    environment {
        NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO = credentials('nexus-repo')
    }
    stages {
        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }   
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Build Artifact') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Push Artifact to Nexus Repo') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic',
                classifier: '',
                file: 'target/spring-petclinic-2.4.2.war',
                type: 'war']],
                credentialsId: 'nexus-cred',
                groupId: 'Petclinic',
                nexusUrl: 'nexus.dobetabeta.shop',
                nexusVersion: 'nexus3',
                protocol: 'https',
                repository: 'nexus-repo',
                version: '1.0'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/petclinicapps .'
            }
        }
        stage('Trivy image Scan') {
            steps {
                sh "trivy image $NEXUS_REPO/petclinicapps > trivyimage.txt"
            }
        }
    //     stage('Trivy Image Scan') {
    //         steps {
    //             script {
    //             // Run Trivy scan and save the output in JSON format
    //                 sh 'trivy image --format json -o trivy-image-report.json $NEXUS_REPO/petclinicapps || true'
    //                 // Convert JSON report to HTML
    //                 writeFile file: 'convert_to_html.py', text: '''
    // import json
    // import sys
    // def convert_json_to_html(json_file, html_file):
    //     with open(json_file, 'r') as f:
    //         data = json.load(f)
    //     # Basic HTML conversion, for better formatting consider using libraries like Jinja2
    //     html_content = '<html><head><title>Trivy Image Scan Report</title></head><body>'
    //     html_content += '<h1>Trivy Image Scan Report</h1>'
    //     html_content += '<h2>Summary</h2>'
    //     html_content += '<pre>' + json.dumps(data, indent=4) + '</pre>'
    //     html_content += '</body></html>'
    //     with open(html_file, 'w') as f:
    //         f.write(html_content)
    // if __name__ == "__main__":
    //     if len(sys.argv) != 3:
    //         print("Usage: python convert_to_html.py <input_json> <output_html>")
    //         sys.exit(1)
    //     convert_json_to_html(sys.argv[1], sys.argv[2])
    // '''
    //         // Convert JSON to HTML
    //                 sh 'python3 convert_to_html.py trivy-image-report.json trivy-image-report.html'
    //         // Archive the HTML report as a build artifact
    //                 archiveArtifacts artifacts: 'trivy-image-report.html', allowEmptyArchive: true
    //         // Publish the HTML report
    //                 publishHTML(target: [
    //                     reportDir: '.',
    //                     reportFiles: 'trivy-image-report.html',
    //                     keepAll: true,
    //                     alwaysLinkToLastBuild: true,
    //                     includeInIndex: true,
    //                     indexFilename: 'index.html'
    //                 ])
    //             }
    //         }
    //     }
        // stage('Trivy Image Scan') {
        //     steps {
        //         sh 'echo Analyse with Trivy'
        //         // Run the Trivy image scan and save the output in HTML format
        //         sh "trivy image --format template -o trivy-image-report.html $NEXUS_REPO/petclinicapps"
        //         // Archive the HTML report as a build artifact
        //         archiveArtifacts artifacts: 'trivy-image-report.html', allowEmptyArchive: true
        //         // Publish the HTML report
        //         publishHTML(target: [
        //             reportDir: '.',
        //             reportFiles: 'trivy-image-report.html',
        //             reportName: "Trivy Image CVE Report",
        //             keepAll: true,
        //             alwaysLinkToLastBuild: true,
        //             includeInIndex: true,
        //             indexFilename: 'index.html'
        //         ])
        //     }
        // }
        stage('Log Into Nexus Docker Repo') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
            }
        }
        stage('Push to Nexus Docker Repo') {
            steps {
                sh 'docker push $NEXUS_REPO/petclinicapps'
            }
        }
        stage('Deploy to stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                        ssh -t -t -o StrictHostKeyChecking=no -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@34.254.89.91" ubuntu@10.0.4.42 "ansible-playbook -i /etc/ansible/stage-hosts /etc/ansible/stage-playbook.yml"
                    '''
                }
            }
        }
        stage('check stage website availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://stage.dobetabeta.shop"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://stage.dobetabeta.shop", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The stage petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The stage petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }
        stage('Deploy to prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                        ssh -t -t -o StrictHostKeyChecking=no -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@34.254.89.91" ubuntu@10.0.4.42 "ansible-playbook -i /etc/ansible/prod-hosts /etc/ansible/prod-playbook.yml"
                    '''
                }
            }
        }
        stage('check prod website availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://prod.dobetabeta.shop"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://prod.dobetabeta.shop", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The prod petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The prod petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
    

pipeline {
  agent any
  environment {
    TRIVY_IMAGE = 'your-image:latest' // Replace with your Docker image
  }
  stages {

  }
}