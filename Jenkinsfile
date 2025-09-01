pipeline {
    agent { label 'control_node_agent0' }
    
    parameters {
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: false, description: 'Clean workspace before build')
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
                    params.CLEAN_WORKSPACE || !fileExists('homelab_ansible_playbooks')
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

        stage('[Terraform] Init & Plan') {
            steps {
                dir('homelab_terraform_configs/create_k3s_cluster') {
                    withCredentials([
                        string(credentialsId: 'ci_username',                  variable: 'TF_VAR_ci_username'),
                        string(credentialsId: 'ci_password',                  variable: 'TF_VAR_ci_password'),
                        string(credentialsId: 'pm_api_endpoint',              variable: 'TF_VAR_pm_api_endpoint'),
                        string(credentialsId: 'pm_api_token',                 variable: 'TF_VAR_pm_api_token'),
                        string(credentialsId: 'ssh_auxilery_public_key',      variable: 'TF_VAR_ssh_auxilery_public_key'),
                        string(credentialsId: 'terraform_ssh_user',           variable: 'TF_VAR_terraform_ssh_user'),
                        string(credentialsId: 'ssh_ansible_public_key',       variable: 'TF_VAR_ssh_ansible_public_key'),
                        file(credentialsId: 'terraform_ssh_private_key_file', variable: 'SSH_KEY_FILE'),
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
                        string(credentialsId: 'ci_username',                  variable: 'TF_VAR_ci_username'),
                        string(credentialsId: 'ci_password',                  variable: 'TF_VAR_ci_password'),
                        string(credentialsId: 'pm_api_endpoint',              variable: 'TF_VAR_pm_api_endpoint'),
                        string(credentialsId: 'pm_api_token',                 variable: 'TF_VAR_pm_api_token'),
                        string(credentialsId: 'ssh_auxilery_public_key',      variable: 'TF_VAR_ssh_auxilery_public_key'),
                        string(credentialsId: 'terraform_ssh_user',           variable: 'TF_VAR_terraform_ssh_user'),
                        string(credentialsId: 'ssh_ansible_public_key',       variable: 'TF_VAR_ssh_ansible_public_key'),
                        file(credentialsId: 'terraform_ssh_private_key_file', variable: 'SSH_KEY_FILE'),
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
                    withCredentials([
                        file(credentialsId: 'ansible_private_key', variable: 'ANSIBLE_PRIVATE_KEY_FILE'),
                    ]) {
                        sh """
                            ansible-playbook k3s_cluster/playbooks/setup_vms.yaml \
                            --private-key "${keyPath}" \
                        """
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
