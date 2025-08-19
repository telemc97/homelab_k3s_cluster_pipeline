pipeline {
    agent { label 'control_node_agent0' }
    
    parameters {
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: true, description: 'Clean workspace before build')
    }
    
    stages {
        
        stage('Cleanup Workspace') {
            when {
                expression { params.CLEAN_WORKSPACE }
            }
            steps {
                cleanWs()
            }
        }

        stage('Checkout Terraform Configs') {
            when {
                expression {
                    params.CLEAN_WORKSPACE || !fileExists('homelab_terraform_configs')
                }
            }
            steps {
                dir('homelab_terraform_configs') {
                    git branch: 'master',
                        url: 'https://gitea.tilhouse.duckdns.org/telemc/homelab_terraform_configs.git',
                        credentialsId: 'gitea_credentials'
                }
            }
        }

        stage('Checkout Ansible Playbooks') {
            when {
                expression {
                    params.CLEAN_WORKSPACE || !fileExists('homelab_terraform_configs')
                }
            }
            steps {
                dir('homelab_ansible_playbooks') {
                    git branch: 'master',
                        url: 'https://gitea.tilhouse.duckdns.org/telemc/homelab_ansible_playbooks.git',
                        credentialsId: 'gitea_credentials'
                }
            }
        }

        stage('Generate SSH Key Pair') {
            steps {
                script {
                    def sshDir = "${env.WORKSPACE}/.ssh"
                    def privateKeyPath = "${sshDir}/id_rsa"
                    def publicKeyPath  = "${sshDir}/id_rsa.pub"

                    // Create .ssh directory and generate SSH key
                    sh """
                        mkdir -p ${sshDir}
                        ssh-keygen -t rsa -b 4096 -f ${privateKeyPath} -N ''
                    """

                    // Read keys and set environment variable
                    def publicKey = readFile(publicKeyPath).trim()
                    def privateKey = readFile(privateKeyPath).trim()

                    env.TF_VAR_ssh_ansible_public_key = publicKey
                    env.ANSIBLE_PRIVATE_KEY = privateKey
                    env.ANSIBLE_PRIVATE_KEY_PATH = privateKeyPath

                    // Secure file permission
                    sh "chmod 600 ${privateKeyPath}"
                }
            }
        }

        stage('[Terraform] Init & Plan') {
            steps {
                dir('homelab_terraform_configs/create_k3s_cluster') {
                    withCredentials([
                        string(credentialsId: 'ci_username',                       variable: 'TF_VAR_ci_username'),
                        string(credentialsId: 'ci_password',                       variable: 'TF_VAR_ci_password'),
                        string(credentialsId: 'pm_api_endpoint',                   variable: 'TF_VAR_pm_api_endpoint'),
                        string(credentialsId: 'pm_api_token',                      variable: 'TF_VAR_pm_api_token'),
                        string(credentialsId: 'ssh_auxilery_public_key',           variable: 'TF_VAR_ssh_auxilery_public_key'),
                        string(credentialsId: 'terraform_ssh_user',                variable: 'TF_VAR_terraform_ssh_user'),
                        file(credentialsId: 'terraform_ssh_private_key_file',      variable: 'SSH_KEY_FILE'),
                    ]){
                        sh 'terraform init'
                        sh '''
                            export TF_VAR_terraform_ssh_private_key="$(cat $SSH_KEY_FILE)"
                            terraform plan \
                            -out=tfplan \
                            -var-file=../common/global_variables.tfvars \
                        '''
                    }
                }
            }
        }

        stage('[Terraform] Apply') {
            steps {
                dir('homelab_terraform_configs/create_k3s_cluster') {
                    withCredentials([
                        string(credentialsId: 'ci_username',                       variable: 'TF_VAR_ci_username'),
                        string(credentialsId: 'ci_password',                       variable: 'TF_VAR_ci_password'),
                        string(credentialsId: 'pm_api_endpoint',                   variable: 'TF_VAR_pm_api_endpoint'),
                        string(credentialsId: 'pm_api_token',                      variable: 'TF_VAR_pm_api_token'),
                        string(credentialsId: 'ssh_auxilery_public_key',           variable: 'TF_VAR_ssh_auxilery_public_key'),
                        string(credentialsId: 'terraform_ssh_user',                variable: 'TF_VAR_terraform_ssh_user'),
                        file(credentialsId: 'terraform_ssh_private_key_file',      variable: 'SSH_KEY_FILE'),
                    ]) {
                        sh '''
                            export TF_VAR_terraform_ssh_private_key="$(cat $SSH_KEY_FILE)"
                            terraform apply \
                            -auto-approve tfplan \
                        '''
                    }
                }
            }
        }

        stage('[Ansible] Provisioning') {
            steps {
                dir('homelab_ansible_playbooks') {
                    script {
                        def keyPath = "${env.WORKSPACE}/.ssh/id_rsa"
                        writeFile file: keyPath, text: env.ANSIBLE_PRIVATE_KEY
                        sh "chmod 600 ${keyPath}"

                        sh """
                            ansible-playbook k3s_cluster/playbooks/setup_vms.yaml \
                            --private-key "${keyPath}" \
                        """
                    }
                }
            }
        }
    }

    post {
        cleanup {
            sh 'rm -rf ${WORKSPACE}/.ssh'
        }
    }
}
