@Library('opt-shared-libraries') _
env.GITLAB_API_URL = "https://gitlab.intranet.opt/api/v4"
env.artifactoryBaseUrl = "https://artifactory.intranet.opt/artifactory/"
env.BRANCH = env.BRANCH_NAME.replaceAll('/', '-')
env.TARGET_BRANCH = "master"
env.GIT_PROJECT_ID = ""
env.GIT_MR_IID = ""
env.GIT_MR_TITLE = "WIP: ${env.JOB_NAME}"
env.PACKAGE_EXT = "zip"
env.PROJECT_NAME = "logstash"
env.GROUP = "nc.opt.elk"
env.ANSIBLE_ID = "308"
env.towerTemplateUrl = "https://tower.intranet.opt/api/v1/job_templates/${env.ANSIBLE_ID}/launch/"
env.GIT_PROJECT_ID = "787"
env.GIT_MR_IID = ""

pipeline {
    agent any
    options {
        gitLabConnection('Gitlab')
        //On n'autorise pas les jobs en parallèle
        disableConcurrentBuilds()
        // Set max number of builds to keep
        buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '15', numToKeepStr: '10'))
        //On affiche les timestamps
        timestamps()
    }
    stages {
        stage('checkout') {
            steps {
                checkout scm
            }
        }
        stage('merge request') {
            when {
                not {
                    expression {
                        return env.BRANCH_NAME == "master"
                    }
                }
            }
            steps {
                script {
                    if (branch != "master") {
                        env.GIT_MR_IID = createMRAndGetId(env.GITLAB_API_URL, env.GIT_PROJECT_ID)
                        //Get the Merge Request IID if the MR has already been created and its a replayed job
                        if (env.GIT_MR_IID == "") {
                            env.GIT_MR_IID = getMRId(env.GITLAB_API_URL, env.GIT_PROJECT_ID)
                        }
                    }
                }
            }

        }
        stage('merge') {
            steps {
                mergeToTargetBranch()
            }
        }
        stage('packaging') {
            steps {
                script{
                    script{
                        def information = readProperties file: 'information.properties'
                        def fileName = "${env.PROJECT_NAME}-${information.version}.${env.PACKAGE_EXT}"
                        zip dir: 'logstash', zipFile: fileName
                    }
                }
            }
        }
        stage('publish') {
            when {
                environment name: 'BRANCH_NAME', value: 'master'
            }
            steps {
                script {
                    def server = Artifactory.server 'ART'
                    def information = readProperties file: 'information.properties'
                    def groupPath = env.GROUP.replaceAll("\\.", "/")
                    def packagePath = "script-release-local/${groupPath}/${env.PROJECT_NAME}/${information.version}/${env.PROJECT_NAME}-${information.version}.${env.PACKAGE_EXT}"
                    def uploadSpec = """ {
                         "files": [
                         {
                             "pattern": "${env.PROJECT_NAME}-${information.version}.${env.PACKAGE_EXT}",
                             "target": "${packagePath}",
                             "props": "source=jenkins",
                             "regexp": true
                         }
                         ]
                     }"""

                    def buildInfo = server.upload(uploadSpec)
                    server.publishBuildInfo(buildInfo)
                }
            }
        }
        stage('deploy') {
            when {
                environment name: 'BRANCH_NAME', value: 'master'
            }
            steps {
                script {
                    def information = readProperties file: 'information.properties'
                    def groupPath = env.GROUP.replaceAll("\\.", "/")
                    def packagePath = "script-release-local/${groupPath}/${env.PROJECT_NAME}/${information.version}/${env.PROJECT_NAME}-${information.version}.${env.PACKAGE_EXT}"
                    def warUrl = "${env.artifactoryBaseUrl}${packagePath}"
                    def json = '''{
                          "extra_vars": {
                            "self_url": "''' + warUrl + '''"
                          }
                        }'''

                    withCredentials([usernamePassword(credentialsId: 'jenkins_curlbot', usernameVariable: 'USER', passwordVariable: 'PWD')]) {
                        sh "curl --noproxy '*' -vv -k -H 'Content-Type: application/json' -XPOST --user ${USER}:${PWD} -d '${json}' ${env.towerTemplateUrl}"
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                if (env.BRANCH_NAME != "master") {
                    def GIT_MR_COMMENT = ":white_check_mark: Jenkins Build \n\nResults available at: [Jenkins [$env.JOB_NAME#$env.BUILD_NUMBER]]($env.BUILD_URL)"
                    withCredentials([usernamePassword(credentialsId: 'b9ae5e26-b47f-4af1-94bd-e9d95b9cebcc', usernameVariable: 'USER', passwordVariable: 'PWD')]) {
                        sh "curl -w '%{http_code}' -o /dev/null --noproxy '*' -k --header \"PRIVATE-TOKEN: ${PWD}\" -X POST \"${env.GITLAB_API_URL}/projects/${env.GIT_PROJECT_ID}/merge_requests/${env.GIT_MR_IID}/notes\" --data \"body=${GIT_MR_COMMENT}\" > result"
                    }
                    updateGitlabCommitStatus name: 'jenkins', state: 'success'
                }
                deleteDir()
            }
        }
        failure {
            script {
                if (env.BRANCH_NAME != "master") {
                    def GIT_MR_COMMENT = ":x: Jenkins Build \n\nResults available at: [Jenkins [$env.JOB_NAME#$env.BUILD_NUMBER]]($env.BUILD_URL)"
                    withCredentials([usernamePassword(credentialsId: 'b9ae5e26-b47f-4af1-94bd-e9d95b9cebcc', usernameVariable: 'USER', passwordVariable: 'PWD')]) {
                        sh "curl -w '%{http_code}' -o /dev/null --noproxy '*' -k --header \"PRIVATE-TOKEN: ${PWD}\" -X POST \"${env.GITLAB_API_URL}/projects/${env.GIT_PROJECT_ID}/merge_requests/${env.GIT_MR_IID}/notes\" --data \"body=${GIT_MR_COMMENT}\" > result"
                    }
                    updateGitlabCommitStatus name: 'jenkins', state: 'failed'
                }
                deleteDir()
            }
        }
    }
}

