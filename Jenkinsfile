node {
  def goPath = "/go"
  def workDir = "${goPath}/src/github.com/lachie83/croc-hunter/"
  def pwd = pwd()
  def dockerEmail = "."
  def quay_creds_id = "quay_creds"

  checkout scm

  // read in required jenkins workflow config values
  def inputFile = readFile('Jenkinsfile.json')
  def config = new groovy.json.JsonSlurperClassic().parseText(inputFile)
  println "pipeline config ==> ${config}"

  // continue only if pipeline enabled
  if (!config.pipeline.enabled) {
      println "pipeline disabled"
      return
  }

  // load pipeline library
  dir('lib/jenkins-pipeline') {
      git branch: config.pipeline.library.branch,
              url: 'https://github.com/lachie83/jenkins-pipeline.git'
  }

  stage ('preparation') {

  sh "wget http://storage.googleapis.com/kubernetes-helm/helm-v2.0.0-alpha.5-linux-amd64.tar.gz -O /tmp/helm.tar.gz"
  sh "tar -C /tmp -xvzf /tmp/helm.tar.gz"
  sh "cp /tmp/linux-amd64/helm /usr/local/linux-amd64/helm"

  sh "env | sort"

  sh "mkdir -p ${workDir}"
  sh "cp -R ${pwd}/* ${workDir}"

  }

  stage ('compile') {

  sh "cd ${workDir}"
  sh "make bootstrap build"
  sh "go test -v -race ./..."

  }

  stage ('lint') {

  sh "/usr/local/linux-amd64/helm lint ${pwd}/charts/croc-hunter"

  }

  stage ('publish') {

    withCredentials([[$class          : 'UsernamePasswordMultiBinding', credentialsId: quay_creds_id,
                    usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {

    sh "echo ${env.PASSWORD} | base64 --decode > ${pwd}/docker_pass"
    sh "docker login -e ${dockerEmail} -u ${env.USERNAME} -p `cat ${pwd}/docker_pass` quay.io"
      
      sh "cd ${pwd}"
      sh "make docker_build"
      sh "make docker_push"
      }
  }

  stage ('deploy') {

  // start kubectl proxy to enable kube API access

  // TOOD: Load in pipeline functions to handle this
  // def pipeline = load("lib/jenkins-pipeline/pipeline.groovy")
  // pipeline.kubectlProxy()

  def name = "croc-hunter"
  def replicas = "3"
  def cpu = "10m"
  def memory = "128Mi"

  sh "kubectl proxy &"
  sh "sleep 5"
  sh "kubectl --server=http://localhost:8001 get nodes"

  sh "/usr/local/linux-amd64/helm init"

  sh "/usr/local/linux-amd64/helm upgrade croc-hunter ${pwd}/charts/croc-hunter --set ImageTag=${env.BUILD_NUMBER},Replicas=${replicas},Cpu=${cpu},Memory=${memory} --install --namespace croc-hunter"

  }
}
