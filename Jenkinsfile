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
docker build -t hemanth .
docker tag hemanth:latest amukrishna05/hemanth:latest
docker push amukrishna05/hemanth:latest '''
}
}
stage ("Linting Dockerfile") {
steps {
sh 'hadolint ./Dockerfile | tee -a dockerfile_lint.txt'
}
post {
always {
 archiveArtifacts 'dockerfile_lint.txt'
}
}
}
stage ('trivyscan') {
steps {
script {
sh '''
image_name=hemanth:latest
echo $image_name
apk --no-cache add curl || true
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl > html.tpl
trivy image --ignore-unfixed --format template --template "@html.tpl" -o cve_report.html ${image_name}
mkdir publish
cp -r cve_report.html $WORKSPACE/publish
'''
publishHTML([
allowMissing: false,
 alwaysLinkToLastBuild: false,
keepAll: false,
reportDir: '$WORKSPACE/publish',
reportFiles: 'cve_report.html',
reportName: 'Trivy Scan',
reportTitles: 'Trivy Scan'
])
}
}
}
stage ('deployment') {
steps {
withCredentials([string(credentialsId: 'kubernetes', variable: 'KUBE_API_TOKEN')]) {
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
