readProperties = loadConfigurationFile 'buildConfiguration'
pipeline {
  agent { label 'swarm'}
  environment {
      TOKEN = credentials('gh-token')
      TF_PLUGIN_CACHE_DIR = '/plugins'
  }
  triggers { pollSCM('H/5 * * * *') }
  stages {
    stage('Foodcritic'){
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        sh 'foodcritic -B cookbooks/collectd/ || exit 0'
      }
      post{
        always {
          warnings canComputeNew: false, canResolveRelativePaths: false, categoriesPattern: '', consoleParsers: [[parserName: 'Foodcritic']], defaultEncoding: '', excludePattern: '', healthy: '100', includePattern: '', messagesPattern: '', unHealthy: ''
        }
      }
    }
    stage('Rubocop'){
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        sh 'sudo su --command "/opt/chefdk/embedded/bin/rubocop –L cookbooks/collectd/ -r rubocop/formatter/checkstyle_formatter -f RuboCop::Formatter::CheckstyleFormatter -o int-lint-results.xml" || exit 0'
      }
      post{
        always {
          checkstyle canComputeNew: false, canRunOnFailed: true, defaultEncoding: '', healthy: '', pattern: 'int-lint-results.xml', unHealthy: ''
        }
      }
    }
    stage('ChefSpec'){
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        sh """
            cd cookbooks/collectd/
            chef exec rspec
        """
      }
    }
    stage("Kitchen test"){
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        script {
          kitchenParallel (this.getInstances(), "collectd")
        }
      }
    }
    stage("Approval step"){
      agent none
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        input message: "Do you want to create a PR to master branch?", ok: 'Approve'
      }
    }
    stage('Generate PR'){
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        createPR "jenkinsdou", readProperties.title, "master", env.BRANCH_NAME, "mons3rrat"
        slackSend baseUrl: readProperties.slack, channel: '#docker-anniversary', color: '#00FF00', message: "Please review and approve PR to merge changes to dev branch : https://github.com/mons3rrat/tl_chef_pipeline/pulls"
        }
    }
    stage('Knife cookbook upload'){
      when { expression{ env.BRANCH_NAME == 'master'} }
      steps{
        sh "knife cookbook upload collectd -V --config-option cookbook_path='cookbooks/'"
      }
    }
  }
  post {
    success {
      slackSend baseUrl: readProperties.slack, channel: '#docker-anniversary', color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
    }
    failure {
      script{
        def commiter_user = sh "git log -1 --format='%ae'"
        slackSend baseUrl: readProperties.slack, channel: '#docker-anniversary', color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
      }
    }
  }
}

