#!groovy
//V2

def appName = "gem-business-calendar"
def rubyVersion = "2.5.3"
def main = new io.youse.main()
def deployStage = false

slackSend channel: '#deploys', message: "Build Started - ${env.JOB_NAME} #${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"

stage("code-pull") {
  node('jenkins-qa-stage') {
    sh "sudo chown -R jenkins.jenkins ${env.WORKSPACE}"
    deleteDir()
    checkout scm

    stash name: "${env.BUILD_TAG}", useDefaultExcludes: false
  }
  main.githubProject('business_calendar', '10', true)
}

stage("unit-tests") {
  waitUntil {
    try {
      node('jenkins-qa-stage') {
        gemCacheDir = "/data/gem-cache/${appName}"
        sh "mkdir -p ${gemCacheDir}"

        docker.image("youse-rpm:${rubyVersion}").inside("--net=isolated_docker \
          -v ${gemCacheDir}:${env.WORKSPACE}/vendor/bundle \
          --net-alias=${appName}-${env.BUILD_TAG} \
          --name=${env.BUILD_TAG}") {

          withEnv(["HOME=${env.WORKSPACE}"]) {
            ansiColor('xterm') {
              sh '''
              bundle install --path vendor/bundle
              bundle exec rspec --fail-fast --colour --tty
              '''
            }
          }
        }
      }
      return true

    } catch(e) {
      echo "${e}"
      echo "Fail! Rspec tests"
      main.retry("#deploys", deployStage)
    }
  }
}

stage("check-version") {
  node('jenkins-qa-stage') {
    version = main.get_gem_version('lib/business_calendar/version.rb')
    main.check_gem_version(version)
  }
}

if (currentBuild.result == "UNSTABLE") {
  return
}

stage("build-gem") {
  waitUntil {
    try {
      node('jenkins-qa-stage') {
        main.build_gem(appName,rubyVersion,version)
      }
      return true

    } catch(e) {
      echo "${e}"
      main.retry("#deploys", deployStage)
    }
  }
}
