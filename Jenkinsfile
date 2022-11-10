String buildxContainer = 'buildx-container'
String webhooks = 'webhooks'

// Add parameters HERE.
List customParameters = []

library(
  identifier: 'jenkins-shared-lib@v6.0.3',
  retriever: modernSCM([
    $class: 'GitSCMSource',
    remote: 'https://github.com/Tealium/jenkins-shared-lib',
    credentialsId: 'github-cicd-bot-teal-token'
  ])
)

pipeline {
  agent {
    kubernetes {
      yamlFile 'jenkins/k8s-pods.yml'
    }
  }

  options {
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '15'))
    skipStagesAfterUnstable()
    timeout(time: 1, unit: 'HOURS')
    timestamps()
  }

  // Environment variables within this top-level environment block can be seen across all containers in any stage.
  environment {
    // Common metadata used across every pipeline and many containers and helper scripts.
    ACCOUNT_ID = "${env.ACCOUNT_ID}"
    ACCOUNT_NAME = "${env.ACCOUNT_NAME}"
    AWS_MAX_ATTEMPTS = '20'
    ENVIRONMENT = "${env.ENVIRONMENT}"
    ENVIRONMENT_PREFIX = "${env.ENVIRONMENT_PREFIX ?: ''}"
    ENVIRONMENT_TYPE = "${env.ENVIRONMENT_TYPE}"
    PLATFORM_NAME = "${env.PLATFORM_NAME}"
    PREFIXED_ENVIRONMENT = "${env.ENVIRONMENT_PREFIX ?: ''}${env.ENVIRONMENT}"
    REGION = "${params.REGION_OVERRIDE ?: env.REGION}"

    // Place pipeline-specific variables here.
    COMPONENT = "${env.JOB_NAME}"
    JFROG_CLI_OFFER_CONFIG = "${false}"
  }

  stages {
    // CICD-related Jenkins pipeline configuration. 'Update Pipeline' MUST be the first stage in the pipeline.
    stage('Update Pipeline') {
      steps { script { updatePipeline(customParameters) } }
    }

    stage('Git Checkout') {
      steps { container(webhooks) { script { gitCheckout() } } }
    }

    stage('Send Webhooks') {
      steps {
        container(webhooks) {
          script {
            sourceVersion()
            webhooksHelper.inProgress()
            sourceVersion.checkUpToDate()
          }
        }
        buildName "#${env.BUILD_NUMBER}_${sourceVersion.runningOnDefaultBranch() ? 'Release' : 'Snapshot'}"
      }
    }

    stage('Check for Artifact') {
      steps {
        container(buildxContainer) {
          script {
            // Currently code-climate only supports amd64, so we can only include it if that is the only arch we build
            jfrogArtifact.platforms = ['linux/amd64']
            jfrogArtifact()
          }
        }
      }
    }

    stage('Build and Publish') {
      when { not { expression { jfrogArtifact.exists() } } }
      steps {
        container(buildxContainer) {
          script {
            jfrogArtifact.dockerBuildAndPush()
          }
        }
      }
    }

    // CICD-related. Should be final stage.
    stage('Create Pre-Release') {
      when { expression { sourceVersion.runningOnDefaultBranch() } }
      steps { container(webhooks) { script { cutRelease() } } }
    }
  }

  post {
    always {
      container(webhooks) {
        script {
          deploymentOrchestrator(env.PLATFORM_NAME)
          sourceVersion.checkUpToDate()
          webhooksHelper.post()
        }
      }
    }
  }
}
