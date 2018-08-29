#!/usr/bin/groovy
@Library('github.com/stakater/fabric8-pipeline-library@v2.6.5')

def dummy

toolsNode(toolsImage: 'stakater/pipeline-tools:1.13.2') {
  container(name: "tools") {
    withCurrentRepo { def repoUrl, def repoName, def repoOwner, def repoBranch ->

      def utils = new io.fabric8.Utils()
      def common = new io.stakater.Common()
      def slack = new io.stakater.notifications.Slack()
      def git = new io.stakater.vc.Git()

      // Slack variables
      def slackChannel = "${env.SLACK_CHANNEL}"
      def slackWebHookURL = "${env.SLACK_WEBHOOK_URL}"

      def modulesDirectory = "modules"
      
      try {

        stage('Validate Modules') {
          sh """
            for dir in ${modulesDirectory}/*/
            do
              dir=\${dir%*/}
              terraform init \${dir}
              terraform validate \${dir}
            done
          """
        }

        if(utils.isCD()) {
          stage('Tag and Release') {
            print "Generating New Version"
            
            def versionFile = ".version"
            def version = common.shOutput("jx-release-version --gh-owner=${repoOwner} --gh-repository=${repoName} --version-file ${versionFile}")

            // Save new version
            sh """
              echo "${version}" > ${versionFile}
            """

            // Notify on JIRA
            sh """
                stk notify jira --comment "Version ${version} of ${repoName} has been successfully built and released."
            """

            git.commitChanges(WORKSPACE, "Bump Version to ${version}")

            print "Pushing Tag ${version} to Git"
            git.createTagAndPush(WORKSPACE, version)
            git.createRelease(version)
          }
        }

        stage('Notify') {
          def slackFields = []
          if (utils.isCD()) {
            slackFields = [slack.createField("version", version, true)]
          }

          slack.sendDefaultSuccessNotification(slackWebHookURL, slackChannel, slackFields)

          def commentMessage = "Terraform modules validated successfully!"
          git.addCommentToPullRequest(commentMessage)
        }

      }
      catch(e) {
        slack.sendDefaultFailureNotification(slackWebHookURL, slackChannel, [slack.createErrorField(e)])

        def commentMessage = "Yikes! You better fix it before anyone else finds out! [Build ${env.BUILD_NUMBER}](${env.BUILD_URL}) has Failed!"
        git.addCommentToPullRequest(commentMessage)
        sh """
            stk notify jira --comment "${commentMessage}"
        """
        throw e
      }
    }
  }
}