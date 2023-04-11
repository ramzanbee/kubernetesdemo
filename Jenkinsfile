pipeline {
agent any
environment {
DOCKER = credentials('dockercreds')
CERT = "$WORKSPACE/cert/${cert_name}"
}
parameters {
string(name: 'KUBE_API_EP', defaultValue: 'https://demo-nlb-9e7388e9858d260f.elb.ap-northeast-1.amazonaws.com', description: 'provide the image tag to be deployed')
string(name: 'cert_name', defaultValue: 'k8s-deploy.crt', description: 'Add/Select the K8s CERT filename of the cluster you want to connect')
}
stages {
stage ('git-checkout') {
steps {
cleanWs()
git 'https://github.com/ramzanbee/kubernetesdemo.git'
}
}
stage ('maven-build') {
steps {
sh '''
mvn clean package '''
}
}
stage ('docker-build') {
steps {
sh '''
docker login -u $DOCKER_USR -p $DOCKER_PSW
cd $WORKSPACE
docker build -t demo1 .
docker tag demo1:latest ramzanbee/demo1:latest
docker push ramzanbee/demo1:latest '''
}
}
stage ("Linting Dockerfile") {
agent {
docker {
image 'pipelinecomponents/hadolint'
reuseNode true
}
}
steps {
sh 'hadolint ./Dockerfile | tee -a dockerfile_lint.txt'
}
post {
always {
 archiveArtifacts 'dockerfile_lint.txt'
}
}
}
stage ('deployment') {
steps {
withCredentials([string(credentialsId: 'K8S-API-TOKEN', variable: 'KUBE_API_TOKEN')]) {
sh '''
sh $WORKSPACE/cert/set-k8s-context.sh $KUBE_API_EP $KUBE_API_TOKEN $CERT
kubectl get nodes --insecure-skip-tls-verify
kubectl apply -f regapp-deploy.yml --insecure-skip-tls-verify
kubectl apply -f regapp-service.yml --insecure-skip-tls-verify
kubectl get pods -o wide --insecure-skip-tls-verify
kubectl get nodes -o wide --insecure-skip-tls-verify
ls -la
'''
}
}
}
}
post {
always {
cleanWs()
}
}
}
