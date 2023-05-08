pipeline {
	agent any

	environment {
		PATH = "${PATH}:${getTerraformPath()}"
		AWS_DEFAULT_REGION='us-east-1'
		AWS_CREDENTIALS = credentials('aws-auth')
		KUBECONFIG="${KUBECONFIG}:${HOME}/.kube/config"
	}

	stages {

		stage("Create S3 Bucket") {
			steps{
				script{
					createS3Bucket('raycoy-aws-deploy-jenkins')
				}
			}
		}

		stage("Create DynamoDB") {
			steps{
				script{
					createDynamoDB('raycoy-aws-deploy-jenkins-lock')
				}
			}
		}

		stage("Create ECR_REPO") {
			steps{
				script {
					createECR('aws-deploy-cicd-repo')
				}
			}
		}

		stage("Terraform Init & Apply") {
			steps{
				dir('terraform/ecr_registry') {
                    sh "pwd"
                    // sh returnStatus: true, script: 'terraform workspace new dev'
                    sh "terraform init -upgrade"
                    sh "terraform apply -auto-approve -var-file=dev.tfvars"
                    script{
                        registry_id = sh(returnStdout: true, script: "terraform output registry_id").trim()
                        repository_name = sh(returnStdout: true, script: "terraform output repository_name").trim()
                        repository_url = sh(returnStdout: true, script: "terraform output repository_url").trim()
                    }
                }
			}
		}

		stage("AWS ECR Authentication") {
			steps{
				sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${registry_id}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
			}
		}

		stage("Deploy App Image") {
			steps{
				sh "docker image build -t ${repository_name}:${BUILD_ID} ."
                sh "docker tag ${repository_name}:${BUILD_ID} ${repository_url}:${BUILD_ID}"
                sh "docker push ${repository_url}:${BUILD_ID}"
				sh "docker image rmi -f ${repository_name}:${BUILD_ID}"
				sh "docker image rmi -f ${repository_url}:${BUILD_ID}"
			}
		}

		stage("k8s Deployment"){
			steps {
				script {
					withKubeConfig(caCertificate: '', clusterName: 'vangel-app.us-east-1.eksctl.io', contextName: 'kops@vangel-app.us-east-1.eksctl.io', credentialsId: 'jenkins-deployer-credentials', namespace: '', restrictKubeConfigAccess: false, serverUrl: 'https://FCFAE07FA7607413D7749329793EFA63.gr7.us-east-1.eks.amazonaws.com') {
						// sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${registry_id}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
						// NAMESPACE_NAME="health-check" && \
						
						sh returnStatus: true, script: 'kubectl create secret docker-registry regcred --docker-server=${registry_id}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com --docker-username=AWS --docker-password=$(aws ecr get-login-password)'
						// --namespace=$NAMESPACE_NAME || true && \
					
						sh 'kubectl apply -f dnginx.yaml'
						sh 'envsubst < dapps.yaml | kubectl apply -f -'
					}
				}
			}
		}
		

	}

}



def createS3Bucket(bucketName){
    sh returnStatus: true, script: "aws s3 mb s3://${bucketName} --region=us-east-1"
}

def createECR(repoName){
    sh returnStatus: true, script: "aws ecr create-repository --repository-name ${repoName} --image-scanning-configuration scanOnPush=true --region ${AWS_DEFAULT_REGION}"
}


def createDynamoDB(dynamodbName){
    sh returnStatus: true, script: "aws dynamodb create-table --table-name ${dynamodbName} --attribute-definitions AttributeName=LockID,AttributeType=S --key-schema AttributeName=LockID,KeyType=HASH --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5"
}


def getTerraformPath(){
	def tfHome = tool name: 'terraform:1.4.6', type: 'terraform'
	return tfHome
}


def registry_id

def repository_name

def repository_url