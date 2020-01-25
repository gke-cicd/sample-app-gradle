pipeline {
  agent {
    label "jenkins-gradle"
  }
  environment {
    ORG = 'gke-cicd'
    APP_NAME = 'sample-app-gradle'
    CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
    DOCKER_REGISTRY_ORG = 'gke-cicd'
  }
  stages {
    stage('App. test MVN') {
      when {
        branch 'PR-*'
      }
      steps {
        container('maven') {
          sh "mvn verify"
          // If error the pipeline should send a Github failed check in the PR.
        }
      }
    }
  stage('Sonar Code analysis') {
      when {
        branch 'PR-*'
      }
      steps {
        container('maven') {
          //sh "mvn -Pprod clean verify sonar:sonar"
          // This stage can be executed in parallel with the application tests.
          // If code coverage do not comply with the minimum required the pipeline should send a Github failed check in the PR.
        }
      }
    }
    stage('CI Build and push snapshot') {
      when {
        branch 'PR-*'
      }
      environment {
        PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
        PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
        HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
      }
      steps {
        container('gradle') {
          sh "gradle clean build"
          sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml"
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          dir('./charts/preview') {
            sh "make preview"
            sh "jx preview --app $APP_NAME --dir ../.."
          }
        }
      }
    }
    stage('Build Release') {
      when {
        branch 'master'
      }
      steps {
        container('gradle') {

          // ensure we're not on a detached head
          sh "git checkout master"
          sh "git config --global credential.helper store"
          sh "jx step git credentials"

          // so we can retrieve the version in later steps
          sh "echo \$(jx-release-version) > VERSION"
          sh "jx step tag --version \$(cat VERSION)"
          sh "gradle clean build"
          sh "export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml"
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
        }
      }
    }
    stage('Sonar code analysis in master') {
      when {
        branch 'master'
      }
      steps {
        container('maven') {
         // sh "mvn -Pprod clean verify sonar:sonar"
          // If code coverage do not comply with the minimum required the pipeline should send a Github failed check in the PR.
        }
      }
    }
    stage('Promote to Environments') {
      when {
        branch 'master'
      }
      steps {
        container('gradle') {
          dir('./charts/sample-app-gradle') {
            sh "jx step changelog --version v\$(cat ../../VERSION)"

            // release the helm chart
            sh "jx step helm release"

            // promote through all 'Auto' promotion Environments
            sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
          }
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
