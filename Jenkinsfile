pipeline {
  options {
    timeout(time: 10, unit: 'MINUTES')
  }
  environment {
    PATH_HOME = '/home/jenkins'
  }

  agent {
    node {
      label 'built-in'
      customWorkspace "/home/jenkins/${JOB_NAME}"
    }
  }

  triggers {
    upstream(upstreamProjects: 'grobid/' + BRANCH_NAME.replaceAll('/', '%2F'),\
                                threshold: hudson.model.Result.SUCCESS)
  }

  stages {
    stage('Analyse build cause') {
      steps {
        script {
          analyseBuildCause()
        }
      }
    }
    stage('Build grobid-ner') {
      steps {
        println 'Building grobid-ner'
        script {
          sh 'gradle clean build -x test'
        }
      }
    }
    stage('Publish grobid-ner') {
      steps {
        println 'Publish grobid-ner'
        script {
          sh 'gradle clean publishToMavenLocal'
          // withCredentials([usernamePassword(credentialsId: 'jenkins-artifactory', passwordVariable: 'artifactoryPassword', usernameVariable: 'artifactoryUsername')]) {
          //   script {
          //     sh 'ORG_GRADLE_PROJECT_artifactoryPassword="${artifactoryPassword}" ORG_GRADLE_PROJECT_artifactoryUsername=${artifactoryUsername} gradle publish'
          //   }  
          // }
        }
      }
    }
  }

  post {
    // only triggered when blue or green sign
    success {
      // node is specified here to get an agent
      node('built-in') {
        // keep using customWorkspace to store Junit report
        ws("${PATH_HOME}/${JOB_NAME}") {
          script {
            if (sendEmailNotif("${PATH_HOME}/${JOB_NAME}", "${BUILD_NUMBER}")) {
              println 'sending Success Build notification'
              CUSTOM_SUBJECT = '[CI - Jenkinzz SUCCESS] ' + CUSTOM_SUBJECT
              emailext(
                  mimeType: 'text/html',
                  subject: CUSTOM_SUBJECT,
                  body: '${DEFAULT_CONTENT}',
                  replyTo: '${DEFAULT_REPLYTO}',
                  to: '${ADMIN_RECIPIENTS}' + ';' + CUSTOM_RECIPIENTS
              )
              switchEmailNotif(false, BUILD_NUMBER)
            } else {
              println 'preventing Success Build notification'
            }
          }
        }
      }
    }
    // triggered when red sign
    failure {
      // node is specified here to get an agent
      node('built-in') {
        // keep using customWorkspace to store Junit report
        ws("${PATH_HOME}/${JOB_NAME}") {
          script {
            println 'sending Failure Build notification'
            CUSTOM_SUBJECT = '[CI - Jenkinzz FAILURE] ' + CUSTOM_SUBJECT
            emailext(
                mimeType: 'text/html',
                subject: CUSTOM_SUBJECT,
                body: '${DEFAULT_CONTENT}',
                replyTo: '${DEFAULT_REPLYTO}',
                to: '${ADMIN_RECIPIENTS}' + ';' + CUSTOM_RECIPIENTS
            )
          }
        }
      }
    }
    // triggered when black sign
    aborted {
      println 'post-declarative message: abort job'
    }
    // trigger every-works
    //always {
    //}
  }

  tools {
    jdk 'JDK-1.11'
    gradle 'GRADLE-7.2'
  }
}

// create/remove emailNotif file to trigger email notification
def switchEmailNotif(toggle, build) {
  if (toggle) {
    sh 'echo ' + build + ' > .emailNotif'
  } else {
    if (build == BUILD_NUMBER) {
      sh 'rm -f .emailNotif'
    }
  }
}

// return true if emailNotif file present
boolean sendEmailNotif(path, build) {
  emailNotif = sh(
                 script: "find ${path} -name '.emailNotif'|wc -l",
                 returnStdout: true
               ).trim()
  emailContent = ''
  if (emailNotif == '1') {
    emailContent = sh(
                     script: "cat ${path}/.emailNotif",
                     returnStdout: true
                   ).trim()
  }
  return (emailContent == build)
}

def analyseBuildCause() {
  // Catch if build has been triggered by User
  boolean isStartedByUser = currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause) != null
  if (isStartedByUser) {
    env.SKIP_JOB = '0'
    env.CUSTOM_SUBJECT = JOB_NAME + ' - Manual Build #' + BUILD_NUMBER
    env.CUSTOM_RECIPIENTS = emailextrecipients([[$class: 'RequesterRecipientProvider']])
    switchEmailNotif(true, BUILD_NUMBER)
    println 'Job started by User, proceeding'
  }

  // Catch if build has been triggered by User Commit
  boolean isStartedByCommit = currentBuild.rawBuild.getCause(jenkins.branch.BranchEventCause) != null
  if (isStartedByCommit) {
    env.SKIP_JOB = '0'
    env.CUSTOM_SUBJECT = JOB_NAME + ' - SCM Build #' + BUILD_NUMBER
    env.CUSTOM_RECIPIENTS = emailextrecipients([[$class: 'DevelopersRecipientProvider'], [$class:'CulpritsRecipientProvider']])
    switchEmailNotif(true, BUILD_NUMBER)
    println 'Job started by User Commit, proceeding'
  }

  // Catch if build has been triggered by cron
  boolean isStartedByCron = currentBuild.rawBuild.getCause(hudson.triggers.TimerTrigger$TimerTriggerCause) != null
  if (isStartedByCron) {
    env.SKIP_JOB = '0'
    env.CUSTOM_SUBJECT = JOB_NAME + ' - CRON Build #' + BUILD_NUMBER
    env.CUSTOM_RECIPIENTS = emailextrecipients([[$class: 'DevelopersRecipientProvider'], [$class:'CulpritsRecipientProvider']])
    switchEmailNotif(true, BUILD_NUMBER)
    println 'Job started by Cron, proceeding'
  }

  // Catch if build has been triggered by branch discovery
  boolean isStartedByBranchDiscovery = currentBuild.rawBuild.getCause(jenkins.branch.BranchIndexingCause) != null
  if (isStartedByBranchDiscovery) {
    env.SKIP_JOB = '0'
    env.CUSTOM_SUBJECT = JOB_NAME + ' - BranchDiscovery Build #' + BUILD_NUMBER
    env.CUSTOM_RECIPIENTS = emailextrecipients([[$class: 'DevelopersRecipientProvider'], [$class:'CulpritsRecipientProvider']])
    switchEmailNotif(true, BUILD_NUMBER)
    println 'Job started by Branch Discovery, proceeding'
  }
  // if you want to stop the build without relying on when conditions based on env.SKIP_JOB 
  // currentBuild.result = 'NOT_BUILT'
  // currentBuild.getRawBuild().getExecutor().interrupt(Result.NOT_BUILT)
  // sleep(2)

}
