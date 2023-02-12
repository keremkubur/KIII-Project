pipeline{
    agent any
     environment{
            VERSION = "${env.BUILD_ID}"
        }
    stages{
        stage("sonar quality check"){
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                            sh 'chmod +x gradlew'
                            sh './gradlew sonarqube'
                    }
                     timeout(time: 1, unit: 'HOURS') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                     }

                }
            }
        }
        stage("docker build & docker push"){
                    steps{
                        script{
                            withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                                     sh '''
                                        docker build -t 34.125.34.44:8083/springapp:${VERSION} .
                                        docker logout
                                        docker login -u admin -p $docker_password 34.125.34.44:8083
                                        docker push  34.125.34.44:8083/springapp:${VERSION}
                                        docker rmi 34.125.34.44:8083/springapp:${VERSION}
                                    '''
                            }
                        }
                    }
                }
        stage("pushing the helm charts to nexus"){
                    steps{
                        script{
                            withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                                  dir('kubernetes/') {
                                     sh '''
                                         helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                         tar -czvf  myapp-${helmversion}.tgz myapp/
                                         curl -u admin:$docker_password http://34.125.34.44:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                                    '''
                                  }
                            }
                        }
                    }
                }
        stage('manual approval'){
                    steps{
                         script{
                                  mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> READY FOR DEPLOYMENT <br> Please approve the deployment request: <br> URL: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "DEPLOYMENT APPROVAL: Project name -> ${env.JOB_NAME}", to: "keremkubur1@gmail.com";
                                  input message: 'APPROVE DEPLOYMENT ? ', ok: 'APPROVE'
                                  }

                         }
        }
        stage('deploy application on k8s cluster'){
                	steps{
                	   script{
                	    	withCredentials([kubeconfigFile(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]) {
                			  dir ("kubernetes/"){
                				sh 'helm upgrade --install --set image.repository="34.125.34.44:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ '
                			  }
                		      }
                		    }
                		  }
                		}
    }
    post {
    		always {
    			mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Build URL: ${env.BUILD_URL} <br> For details you can check console output from Jenkins.", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project: ${env.JOB_NAME}", to: "keremkubur1@gmail.com";
    		 }
    	   }
}