#!/usr/bin/groovy
podTemplate(label: 'jenkins-pipeline', containers: [
    containerTemplate(name: 'jnlp', image: 'jenkinsci/jnlp-slave:2.62', args: '${computer.jnlpmac} ${computer.name}', workingDir: '/home/jenkins', resourceRequestCpu: '200m', resourceLimitCpu: '200m', resourceRequestMemory: '256Mi', resourceLimitMemory: '256Mi'),
    containerTemplate(name: 'docker', image: 'docker:1.12.6',       command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:v2.2.2', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.6.1', command: 'cat', ttyEnabled: true)
],
volumes:[
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
]){

  node ('jenkins-pipeline') {
    def pwd = pwd()
    def chart_dir = "${pwd}/k8s"
    def ecr = "405610825889.dkr.ecr.us-east-1.amazonaws.com"
    def app = "nginx-poc"
    def image = "${ecr}/${app}"

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

    // set additional git envvars for image tagging
    println "Setting envvars to tag container"
    sh 'git rev-parse HEAD > git_commit_id.txt'
    try {
      env.GIT_COMMIT_ID = readFile('git_commit_id.txt').trim()
      env.GIT_SHA = env.GIT_COMMIT_ID.substring(0, 7)
    } catch (e) {
      error "${e}"
    }
    println "env.GIT_COMMIT_ID ==> ${env.GIT_COMMIT_ID}"
    def imageTag = "${image}:${env.GIT_COMMIT_ID}"

    // If pipeline debugging enabled
    if (config.pipeline.debug) {
      println "DEBUG ENABLED"
      sh "env | sort"

      println "Runing kubectl/helm tests"
      container('kubectl') {
        println "checking kubectl connnectivity to the API"
        sh "kubectl get nodes"
      }
      container('helm') {
        println "checking client/server version"
        sh "helm version"
      }
    }

    stage ('publish container') {
      container('docker') {
        // build and publish container

        println "Running Docker build/publish: ${imageTag}"
        docker.withRegistry("https://" + ecr, "ecr:us-east-1:ecr-user") {
          def img = docker.image imageTag
          sh "docker build -t ${imageTag}  ."
          img.push()
        }
      }
    }

    // deploy only the master branch
    if (env.BRANCH_NAME == 'master') {
      stage ('deploy to k8s') {
        container('helm') {
          //configure helm client and confirm tiller process is installed
          println "initiliazing helm client"
          sh "helm init"
          // Deploy using Helm chart
          sh "helm dep build ${chart_dir}"
          println "Running deployment"
          sh "helm upgrade --install ${app} ${chart_dir} --set tag=${env.GIT_COMMIT_ID},host=poc.k8s.thredtest.com --namespace=${app}"
          echo "Application ${app} successfully deployed. Use helm status ${app} to check"
        }
      }
    }
  }
}
