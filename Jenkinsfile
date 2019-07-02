@Library('acadiaBuildTools@develop') _
import com.vibrenthealth.jenkinsLibrary.VibrentConstants
def urlBranch
def workspace
def branch
def branchType
def baseVersion
def version
def pullRequest
def label = "worker-${UUID.randomUUID().toString()}"
def pullSecrets = ['reg.vibrenthealth.com', 'dockergroup.vibrenthealth.com']
def npmregistry = "https://nex.vibrenthealth.com/repository/npm"
def vibrentregistry = "https://nex.vibrenthealth.com/repository/vibrent-npm"
def packageJSON
def packageVersion

def dev_branch = "master"

podTemplate(
    cloud:'default', name: label, label:label, imagePullSecrets: pullSecrets,
    containers: kubeUtils.getCiContainers(containerList: ['kubectl', 'sonar-scanner', 'python', 'docker-chrome', 'docker', 'aws-cli-chrome', 'node', 'maven', 'helm']),
    volumes: [
        hostPathVolume(mountPath:'/var/run/docker.sock', hostPath:'/var/run/docker.sock')
    ],
    idleTimeout: 30
) {
    node (label) {
      workspace = pwd()
      branch = env.BRANCH_NAME.replaceAll(/\//, "-")
      branchType = env.BRANCH_NAME.split(/\//)[0]
      urlBranch = env.BRANCH_NAME.replaceAll(/\//, "%252F")
      baseVersion = "${env.BUILD_NUMBER}"
      version = "$branch-$baseVersion"
      env.PROJECT = "keycloak-js-bower"
      if (branch == 'master' || branchType == 'release') {
          env.BRANCH_BUILD = "$branch-$baseVersion"
      }
      def branchCheckout
      pullRequest = env.CHANGE_ID
      if (pullRequest) {
          branchCheckout = "pr/${pullRequest}"
          refspecs = '+refs/pull/*/head:refs/remotes/origin/pr/*'
      }
      else {
          branchCheckout = env.BRANCH_NAME
          refspecs = '+refs/heads/*:refs/remotes/origin/*'
      }
      env.BRANCH = "$branch"


      ciPipeline(
        project:env.PROJECT,
        checkout: {
          stage('Checkout'){
            checkout([
              $class: 'GitSCM',
              branches: [[name: "*/${branchCheckout}"]],
              doGenerateSubmoduleConfigurations: false,
              extensions: [[
                $class: 'SubmoduleOption',
                disableSubmodules  : false,
                parentCredentials  : true,
                recursiveSubmodules: true,
                reference          : '',
                trackingSubmodules : true
              ]],
              submoduleCfg: [],
              userRemoteConfigs: [[
                credentialsId: 'e08f3fab-ba06-459b-bebb-5d7df5f683a3',
                url: 'git@github.com:VibrentHealth/keycloak-js-bower.git',
                refspec: "${refspecs}"
              ]]
            ])
          }
        },
        deploy: {
          container('node') {
            if (branch == 'master' || branchType == 'release') {
              stage('Publish to npm') {
                withCredentials([usernamePassword(credentialsId: VibrentConstants.NEXUS_CREDENTIALS_ID, passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                  sh """
                    npm i -g npm-cli-login
                    NPM_USER="${USER}" NPM_PASS="${PASS}" NPM_EMAIL="mcdev@vibrenthealth.com" NPM_REGISTRY="${vibrentregistry}" npm-cli-login
                  """
                  if (branch == 'master') {
                    sh """
                      # publish
                      npm publish --tag next --tag master --tag latest --registry=${vibrentregistry} --scope=@vibrent
                      """
                  }
                  if (branchType == 'release') {
                    sh """
                      # publish
                      npm publish --tag next --tag latest --registry=${vibrentregistry} --scope=@vibrent
                    """
                  }
                }
              }
            }
          }
        },
        test: {}
      )
    }
  }