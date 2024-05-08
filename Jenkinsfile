def config

pipeline {
    agent any
    stages {
        stage('Pull from github') {
            steps {
                script {
                    config = readYaml file: 'test-change-remote/config.yaml'
                    pull_git(config.sync_git, config.sync_git.repos)
                }
            }
        }

        stage('Push to gitlab') {
            steps {
                script {
                    push_remote_git(config.sync_git, config.sync_git.repos)
                }
            }
        }

        stage('Trigger job') {
            steps {
                script {
                    def trigger_jobs = [:]
                    for (int i = 0; i < config.sync_git.repos.size(); i++) {
                        def item = config.sync_git.repos[i]
                        def check_change = is_change(item)
                        if (check_change == 1 && item.containsKey('trigger_job')) {
                            println("Trigger job ${item.trigger_job}")
                            trigger_jobs["${item.trigger_job}"] = {
                                try {
                                    build job: "${item.trigger_job}/alpha"
                                } catch (hudson.AbortException e) {
                                    println("Cannot trigger job ${item.trigger_job}")
                                }
                            }
                        }
                    }
//                    parallel trigger_jobs
                }
            }
        }
    }
}

def pull_git(config, list_repos) {
    for (int i = 0; i < list_repos.size(); i++) {
        def item = list_repos[i]
        if (fileExists("${item.name}")) {
            sh "rm -rf ${item.name}"
        }
        dir("${item.name}") {
            git(url: "${item.git}", credentialsId: "${item.git_cred}", branch: "${item.git_branch}")
        }
    }

    if (config.containsKey('app_config')) {
        dir("app-config") {
            git(url: "${config.app_config}", credentialsId: "${config.app_config_cred}", branch: "master")
        }
        dir("remote-config") {
            git(url: "${config.remote_config}", credentialsId: "${config.remote_config_cred}", branch: "master")
        }
    }
}

def push_remote_git(config, list_repos) {
    for (int i = 0; i < list_repos.size(); i++) {
        def item = list_repos[i]
        dir("${item.name}") {
            def statusCode = sh(script: 'git remote -v | grep upstream', returnStatus: true)
            if (statusCode == 0) {
                sh 'git remote remove upstream'
            }

            sh "git remote add upstream ${item.remote_git}"

            sshagent(["${item.remote_git_cred}"]) {
                if (item.containsKey('branch_mapping')) {
                    for (int m = 0; m < item.branch_mapping.size(); m++) {
                        def bMap = item.branch_mapping[m]
                        def result = sh(script: "git checkout ${bMap.src}", returnStatus: true)
                        if (result == 0) {
                            sh "git push -uf upstream ${bMap.src}:${bMap.des}"
                        } else {
                            echo "debug: can't checkout to branch ${bMap.src}"
                        }
                    }
                } else {
                    sh "git push -uf upstream alpha"
                }
            }
        }

        if (item.containsKey('configs')) {
            for (int j = 0; j < item.configs.size(); j++) {
                def diff_code = sh(script: "bin/sync_utils.sh diff-config app-config/${item.configs[j].app_config_path} remote-config/${item.configs[j].remote_config_path}", returnStatus: true)
                if (diff_code != 0) {
                    dir("remote-config") {
                        if (fileExists("../app-config/${item.configs[j].app_config_path}")) {
                            echo "Debug: copy config to remote config"
                            if (fileExists("${item.configs[j].remote_config_path}")) {
                                sh "rm -rf ${item.configs[j].remote_config_path}"
                            }
                            sh "mkdir -p ${item.configs[j].remote_config_path}"
                            sh "cp -r ../app-config/${item.configs[j].app_config_path}/* ${item.configs[j].remote_config_path}/"
                            sh "git config user.email \"ntdat09mister@gmail.com\""
                            sh "git config user.name \"ntdat09mister\""
                            sh "git add -A; git commit -m \"update config for service ${item.name} from ${env.JOB_NAME}\""
                            sshagent(["${config.remote_config_cred}"]) {
                                sh "git push origin master"
                            }
                        }
                    }
                }
            }
        }
    }
}

def is_change(repo) {
    def rev = sh(script: "cd ${repo.name} && git rev-parse refs/remotes/origin/${repo.git_branch}", returnStdout: true).trim()
    def last_rev = ""
    def last_commit_file = "${repo.name}.last-commit"
    if (fileExists(last_commit_file)) {
        last_rev = readFile(last_commit_file).trim()
    }

    sh "echo $rev > $last_commit_file"
    if (rev == last_rev) {
        return 0
    }
    return 1
}
