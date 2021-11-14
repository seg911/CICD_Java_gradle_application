pipeline{
    agent any
    environment{
        VERSION = "${env.BUILD_ID}"
    } 
    stages{
        stage("sonar quality check"){
            agent {
                docker {
                    image 'openjdk:11'
                }
            }
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
                               docker build -t nexus.sgm:8083/springapp:${VERSION} .
                               docker run -dp 3000:3000 nexus.sgm:8083/springapp:${VERSION}
                               docker login -u admin -p $docker_password nexus.sgm:8083 
                               docker push  nexus.sgm:8083/springapp:${VERSION}
                               docker rmi nexus.sgm:8083/springapp:${VERSION}
                            '''
                    }
                }
            }
        }
        stage('indentifying misconfigs using datree in helm charts'){
            steps{
                script{

                    dir('kubernetes/') {
                        withEnv(['DATREE_TOKEN=4xB8L87TieLt5JMRwpNLpE']) {
                               sh 'helm datree test myapp/'
                        }
                    }
                }
            }
        }
        stage("pushing the helm chart to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                        dir('kubernetes/') {
                             sh '''
                                helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                tar -czvf myapp-${helmversion}.tgz myapp/
                                curl -u admin:$docker_password http://nexus.sgm:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                        }
                    }
                }
            }
        } 
        stage ('manual approval'){
            when {
            expression { false }
            }
            steps{
                script{
                    timeout(1) {
                      mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Go to build url and approve the deployment: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "segment@mail.sgm";
                      input (id: "Deploy Gate", message: "Deploy ${env.JOB_NAME}?", ok: 'Deploy' ) 
                    }
                }
            }
        }
        stage('Deploying app to kub cluster') {
            steps {
                script{
                  withCredentials([kubeconfigFile(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]) {
                        dir ("kubernetes/"){  
				        sh 'helm list'
				        sh 'helm upgrade --install --set image.repository="nexus.sgm:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ '
                        }
                    }  
                 
                }
            }
        }  
        stage('verify my APP deployment')   {
            steps{
                script{
                   withCredentials([kubeconfigFile(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]){
                     sh 'kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl myjavaapp-myapp:8080'  
                   } 
                }
            }
        }  
    }
    post {
		always {
			mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "segment@mail.sgm";  
		 }
	   }               
}