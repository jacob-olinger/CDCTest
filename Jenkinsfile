@Library('jenkins-library@0.29.0') _

gradleVersion='6.8.1'
gradle="/opt/gradle/gradle-${gradleVersion}/bin/gradle"
ansibleGitLocation="git@gitlab.com:engati/engati-infrastructure/deployment-ansible.git"

appName="product-discovery-engine"

pipeline {
  agent any
  options {
    disableConcurrentBuilds()
    skipDefaultCheckout true
  }

  environment {
    NEXUS_CREDENTIALS = credentials('nexus-artifactory')
  }

  stages {
    stage('Clean WS and checkout SCM') {
      steps {
        deleteDir()
        checkout scm
        script {
          utils.abortPseudoBuild()
        }
      }
    }

	stage('Clean') {
      steps {
        script {
          sh "${gradle} :data-archiver-application:clean -PrepoUser=${NEXUS_CREDENTIALS_USR} -PrepoPassword=${NEXUS_CREDENTIALS_PSW}"
        }
      }
    }

    stage('Increment Version') {
      steps {
        script {
          if (versions.isReleaseBuild()) {
            versions.increment('java')
          }
        }
      }
    }

    stage('Build and Publish') {
      steps {
        script {
          version = versions.getVersion('java')
          sh "${gradle} build install publish -PappVersion=${version} -PrepoUser=${NEXUS_CREDENTIALS_USR} -PrepoPassword=${NEXUS_CREDENTIALS_PSW}"
          artifacts.publishEngati("data-archiver-application/build/libs/${appName}-${version}.jar", "${appName}/${version}")
          artifacts.publishEngati("config/logback-spring.xml", "${appName}/${version}/config")
          artifacts.publishEngati("config/logback-access-spring.xml", "${appName}/${version}/config")
        }
      }
    }

    stage ('Push to Git') {
      steps {
        script {
          if (versions.isReleaseBuild()) {
            sshagent(['jenkins-coviam']) {
              sh "git push origin HEAD:$BRANCH_NAME --tags"
            }
            manifest.publish(
	            appName.replaceAll('-', '_'),
	            ansibleGitLocation,
	            'envs/qa',
	            versions.getVersion('java')
	        )
          }
        }
      }
    }

    stage('Sonarqube') {
      environment {
        scannerHome = tool 'engati-code-scanner'
      }
      steps {
          withSonarQubeEnv('Engati-Sonar') {
            sh "${scannerHome}/bin/sonar-scanner"
        }
      }
    }
  }

  post {
    always {
      script {
        notification.buildStatus("#tech-builds")
      }
      cleanWs()
    }
  }
}