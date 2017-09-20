node("docker") {
  notifyBuild('STARTED')
  pull()

  try {
    runCI()
  } catch (e) {
    // If there was an exception thrown, the build failed
    currentBuild.result = "FAILED"
    throw e
  } finally {
    // Success or failure, always send notifications
    notifyBuild(currentBuild.result)
    sh "docker system prune -f"
  }
}

def runCI() {
  withEnv([
    "COMPOSE_FILE=docker-compose.ci.yml",
    "EMAIL=pat@patscott.io"
  ]) {

    def prodDomain = "cd-example.devopsbliss.com"
    def qaDomain = "cd-example-qa.devopsbliss.com"
    def imageName = "cd-example"
    def stackName = "cd-example"
    def prodTag = "0.${env.BUILD_NUMBER}"
    def devTag = "d${prodTag}"
    def prodLatestTag = "latest"
    def devLatestTag = "d${prodLatestTag}"

    prepare()
    installDevDependencies()
    runTests()
    build(imageName)
    stagingTests()
    tryPublish(imageName, prodTag, prodLatestTag, devTag, devLatestTag)
    release(qaDomain, prodDomain, devTag, prodTag, stackName, imageName)
  }
}

def pull() {
  stage("Pull") {
    checkout scm
  }
}

def prepare() {
  stage("Prepare") {
    // Ensure required files exist
    fileExists 'docker-compose.ci.yml'
    fileExists 'Dockerfile'
    fileExists 'stack.yml'
    fileExists 'package.json'

    // Useful for debugging
    sh "cat docker-compose.ci.yml"
    sh "cat Dockerfile"
    sh "cat Jenkinsfile"
    sh "cat package.json"
    sh "pwd"
    sh "ls -la"
  }
}

def installDevDependencies() {
  stage("Install Dev Dependencies") {
    sh "docker-compose run --rm install"
  }
}

def runTests() {
  stage("Tests") {
    parallel(
      lint: {
        try {
          notifyGithubLint()
          sh "docker-compose run --rm lint"
          notifyGithubLint('SUCCESS')
        } catch(e) {
          notifyGithubLint('FAILURE')
          error "Linting Failed"
        }
      },
      unitTests: {
        try {
          notifyGithubUnit()
          sh "docker-compose run --rm unit-tests"
          // sh "docker-compose run --rm codecov"
          notifyGithubUnit('SUCCESS')
        } catch(e) {
          notifyGithubUnit('FAILURE')
          error "Unit Tests Failed"
        }
      }
    )
  }
}

def build(imageName) {
  stage("Build") {
    withEnv(["NODE_ENV=production"]) {
      sh "docker-compose run --rm build"
      sh "docker-compose run --rm npmClean"
      sh "docker-compose run --rm installProdOnly"
      sh "docker build -t ${imageName} ."
    }
  }
}

def stagingTests() {
  stage("Staging Tests") {
    withEnv(["NODE_ENV=staging"]) {
      prepareStaging()
      runStagingTests()
    }
  }
}

def prepareStaging() {
  sh "docker-compose run --rm clean"
  sh "docker-compose run --rm install"
}

def runStagingTests() {
  try {
    notifyGithubStaging()

    sh "docker-compose up -d staging-deps"
    sh "sleep 10"
    sh "docker-compose run --rm staging"

    notifyGithubStaging('SUCCESS')
  } catch(e) {
    notifyGithubStaging('FAILURE')
    error "Staging failed"
  } finally {
    sh "docker-compose down"
  }
}

def tryPublish(String imageName, String prodTag, String prodLatestTag, String devTag, String devLatestTag) {
  if (env.BRANCH_NAME == 'master') {
    publish(imageName, prodTag, prodLatestTag)
  }

  if (env.BRANCH_NAME == 'dev') {
    publish(imageName, devTag, devLatestTag)
  }
}

def publish(String imageName, String tag, String latestTag) {
  stage("Publish") {
    parallel(
      publishVersion: {
        dockerTagAndPush(imageName, tag)
      },
      publishLatest: {
        dockerTagAndPush(imageName, latestTag)
      }
    )
  }
}

def release(qaDomain, prodDomain, devTag, prodTag, stackName, imageName) {
  if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'dev') {
      sh "docker-compose run --rm install"

      if (env.BRANCH_NAME == 'dev') {
        stage("QA") {
          withEnv(["NODE_ENV=staging"]) {
            deploy(qaDomain, devTag, stackName, imageName, 'last-qa')
          }
        }
      }

      if (env.BRANCH_NAME == 'master') {
        stage("Production") {
          withEnv(["NODE_ENV=production"]) {
            deploy(prodDomain, prodTag, stackName, imageName, 'last-prod')
          }
        }
      }

  }
}

def deploy(domain, tag, stackName, imageName, lastTag) {
  withEnv([
    "SERVICE_DOMAIN=${domain}"
  ]) {
    try {
      withEnv([
        "TAG=${tag}"
      ]) {
        sh "docker stack deploy -c stack.yml qa-${stackName}"
        dockerTagAndPush(imageName, lastTag)
      }

    } catch(e) {
      withEnv([
        "TAG=${lastTag}"
      ]) {
        sh "docker stack deploy -c stack.yml qa-${stackName}"
        error "Deployment to QA has failed"
      }
    }
  }
}

def dockerTagAndPush(imageName, tag) {
  sh "docker tag ${imageName} \
    registry.imakethingsfortheinternet.com/${imageName}:${tag}"
  sh "docker push \
    registry.imakethingsfortheinternet.com/${imageName}:${tag}"
}

def notifyGithubLint(String status = "PENDING") {
  notifyGithub('Lint', 'Ensures interfaces with external dependencies are working properly', status)
}

def notifyGithubUnit(String status = "PENDING") {
  notifyGithub('Unit Tests', 'Ensures interfaces with external dependencies are working properly', status)
}

def notifyGithubStaging(String status = "PENDING") {
  notifyGithub('Staging Tests', 'Ensures interfaces with external dependencies are working properly', status)
}

def notifyGithub(String context, String description, String status = 'PENDING') {  
  githubNotify  credentialsId: "${env.credentialsId}", 
                context: context,
                description: description,
                status: status
}

def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESSFUL'

  // Default values
  def colorName = 'RED'
  def colorCode = '#FF0000'

  def summary = "${buildStatus}: Job `${env.JOB_NAME}` Branch `${env.BRANCH_NAME}` Build `#${env.BUILD_NUMBER}`"
  def message = "${summary} (${env.RUN_DISPLAY_URL})"
  
  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESSFUL') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
  }

  // Send notifications
  slackSend (color: colorCode, message: message)
}