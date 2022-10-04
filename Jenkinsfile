String webhooks = 'webhooks'

// Add parameters HERE.
List customParameters = [ // <<< Local parameters specific to this individual pipeline and not included by default by the shared library parameters.
//   string(
//     name: 'EXAMPLE_CUSTOM_PARAM',
//     defaultValue: 'example value',
//     description: 'example description'
//   ),
]

// CICD-related library of Jenkins pipeline parameters and steps. See jenkins-shared-lib repo.
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

  // DO NOT ADD NEW PARAMETERS HERE in a parameters block. They will be IGNORED.
  // Instead, use the above customParameters list in this pipeline so they can be consumed and updated by the shared library in the 'Update Pipeline' stage.

  // parameters {} // THIS WILL BE IGNORED!

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
    // JFROG_CLI_OFFER_CONFIG = "${false}"  // Required if using jfrog.
    // TF_IN_AUTOMATION = "${true}"         // Required if using terraform.
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
      }
    }

    //
    // Component-specific stages. See jenkins-pipeline-examples repo.
    //
    stage('Build and Upload') {
      steps {
        echo 'Build it upload it'
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
          deploymentOrchestrator(env.ENVIRONMENT)
          sourceVersion.checkUpToDate()
          webhooksHelper.post()
        }
      }
    }
  }
}
