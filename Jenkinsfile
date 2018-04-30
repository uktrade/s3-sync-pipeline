pipeline {

  agent {
    node {
      label env.CI_SLAVE
    }
  }

  options {
    timestamps()
  }

  parameters {
    string(defaultValue: '', description:'Please choose your team: ', name: 'Team')
    string(defaultValue: '', description:'Please choose your project: ', name: 'Project')
    string(defaultValue: '', description:'Please choose your source: ', name: 'Source')
    string(defaultValue: '', description:'Please choose your source bucket: ', name: 'Source_S3_Bucket')
    string(defaultValue: '', description:'Please choose your destination: ', name: 'Destination')
    string(defaultValue: '', description:'Please choose your destination bucket: ', name: 'Destination_S3_Bucket')
    string(defaultValue: '', description:'Please specify your bucket options: ', name: 'S3_Options')
  }

  stages {

    stage('Init') {
      steps {
        script {
          validateDeclarativePipeline("${env.WORKSPACE}/Jenkinsfile")
          deployer = docker.image("ukti/deployer:${env.GIT_BRANCH.split("/")[1]}")
          deployer.pull()
          deployer.inside {
            checkout([$class: 'GitSCM', branches: [[name: env.GIT_BRANCH]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: true, recursiveSubmodules: true, reference: '', trackingSubmodules: false]], submoduleCfg: [], userRemoteConfigs: [[credentialsId: env.SCM_CREDENTIAL, url: env.PIPELINE_SCM]]])
            checkout([$class: 'GitSCM', branches: [[name: env.GIT_BRANCH]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: true, recursiveSubmodules: true, reference: '', trackingSubmodules: false], [$class: 'RelativeTargetDirectory', relativeTargetDir: 'config']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: env.SCM_CREDENTIAL, url: env.PIPELINE_CONF_SCM]]])
            sh "${env.WORKSPACE}/bootstrap.rb"
            options_json = readJSON file: "${env.WORKSPACE}/.ci/option.json"
          }
        }
      }
    }

    stage('Input') {
      steps {
        script {
          input = load "${env.WORKSPACE}/input.groovy"
          if (!env.Team) {
            team = input(
              id: 'team', message: 'Please choose your team: ', parameters: [
              [$class: 'ChoiceParameterDefinition', name: 'Team', description: 'Team', choices: input.get_team(options_json)]
            ])
            env.Team = team
          } else if (!input.validate_team(options_json, env.Team)) {
            error 'Invalid Team!'
          }

          if (!env.Project) {
            project = input(
              id: 'project', message: 'Please choose your project: ', parameters: [
              [$class: 'ChoiceParameterDefinition', name: 'Project', description: 'Project', choices: input.get_project(options_json,team)]
            ])
            env.Project = project
          } else if (!input.validate_project(options_json, env.Team, env.Project)) {
            error 'Invalid Project!'
          }

          if (!env.Source) {
            source = input(
              id: 'source', message: 'Please choose your source: ', parameters: [
              [$class: 'ChoiceParameterDefinition', name: 'Source', description: 'Source', choices: input.get_env(options_json, team, project)]
            ])
            env.Source = source
          } else if (!input.validate_env(options_json, env.Team, env.Project, env.Source)) {
            error 'Invalid Environment!'
          }

          if (!env.Destination) {
            destination = input(
              id: 'destination', message: 'Please choose your destination: ', parameters: [
              [$class: 'ChoiceParameterDefinition', name: 'Destination', description: 'Destination', choices: input.get_env(options_json, team, project)]
            ])
            env.Destination = destination
          } else if (!input.validate_env(options_json, env.Team, env.Project, env.Destination)) {
            error 'Invalid Environment!'
          }
        }
      }
    }

    stage('Sync') {
      steps {
        script {
          ansiColor('xterm') {
            deployer.inside {
              withCredentials([string(credentialsId: env.VAULT_TOKEN_ID, variable: 'TOKEN')]) {
                env.VAULT_SERECT_ID = TOKEN
                sh "${env.WORKSPACE}/bootstrap.rb ${env.Team} ${env.Project} ${env.Source}"
              }
              src_envars = readJSON file: "${env.WORKSPACE}/.ci/env.json"
              src_config = readJSON file: "${env.WORKSPACE}/.ci/config.json"
              sh """
                set +x
                AWS_DEFAULT_REGION=${src_envars.AWS_DEFAULT_REGION} \
                AWS_ACCESS_KEY_ID=${src_envars.AWS_ACCESS_KEY_ID} \
                AWS_SECRET_ACCESS_KEY=${src_envars.AWS_SECRET_ACCESS_KEY} \
                aws s3 sync s3://${env.Source_S3_Bucket} public
              """
            }
            deployer.inside {
              withCredentials([string(credentialsId: env.VAULT_TOKEN_ID, variable: 'TOKEN')]) {
                env.VAULT_SERECT_ID = TOKEN
                sh "${env.WORKSPACE}/bootstrap.rb ${env.Team} ${env.Project} ${env.Destination}"
              }
              dest_envars = readJSON file: "${env.WORKSPACE}/.ci/env.json"
              dest_config = readJSON file: "${env.WORKSPACE}/.ci/config.json"
              sh """
                set +x
                AWS_DEFAULT_REGION=${dest_envars.AWS_DEFAULT_REGION} \
                AWS_ACCESS_KEY_ID=${dest_envars.AWS_ACCESS_KEY_ID} \
                AWS_SECRET_ACCESS_KEY=${dest_envars.AWS_SECRET_ACCESS_KEY} \
                aws s3 sync ${env.S3_Options} public s3://${env.Destination_S3_Bucket}
              """
            }
          }
        }
      }
    }

  }

  post {
    always {
      script{
        deleteDir()
      }
    }
  }

}
