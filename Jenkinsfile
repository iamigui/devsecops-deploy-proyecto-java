pipeline {
  agent any
  environment {
  AWS_ACCOUNT_ID="659026651741"
  AWS_DEFAULT_REGION="us-east-1" 
  IMAGE_REPO_NAME="asg"
  IMAGE_TAG="latest"
  REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
  }
  tools { 
        maven 'Maven_3_5_2'  
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=proyecto -Dsonar.organization=proyectointegrado2023 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=5722676ff9dcd6493ba28d8a3328aec5f8c57885'
			}
    }

	stage('RunSCAAnalysisUsingSnyk') {
            steps {		
				withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
					sh 'mvn snyk:test -fn'
				}
			}
    }		
	stage('Logging into AWS ECR') {
 		steps {
 			script {
 			sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
 			}
 		}
 	}
 
 	// Building Docker images
	stage('Building image') {
 		steps{
 			script {
 			dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
 			}
 		}
 	}
 
 	// Uploading Docker images into AWS ECR
 	stage('Pushing to ECR') {
 		steps{ 
 			script {
 			sh "docker image tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"
 			sh "docker image push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"
 			}
 		}
 	}
	    
	stage('Kubernetes Deployment of ASG Bugg Web Application') {
	   steps {
	      withKubeConfig([credentialsId: 'kubelogin']) {
		  sh('kubectl delete all --all -n devsecops')
		  sh ('kubectl apply -f deployment.yaml --namespace=devsecops')
		}
	      }
   	}

	stage ('wait_for_testing'){
	   steps {
		   sh 'pwd; sleep 180; echo "Application Has been deployed on K8S"'
	   	}
	   }
	   
	stage('RunDASTUsingZAP') {
          steps {
		    withKubeConfig([credentialsId: 'kubelogin']) {
				sh('zap.sh -cmd -quickurl http://$(kubectl get services/webdevops --namespace=devsecops -o json| jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html')
				archiveArtifacts artifacts: 'zap_report.html'
		    }
	     }
       } 
  }
}