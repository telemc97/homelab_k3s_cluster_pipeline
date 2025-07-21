pipeline {
    agent { label 'control_node_agent0' }

        environment {
        TF_VAR_ci_username       = ''
        TF_VAR_ci_password       = ''
        TF_VAR_pm_api_endpoint   = ''
        TF_VAR_pm_api_token      = ''
        TF_VAR_node_public_key   = ''
        TF_VAR_MacBook_public_key = ''
        }

        stages {
            stage('Setup Terraform Env') {
                steps {
                    script {
                    withCredentials([
                        string(credentialsId: 'ci_username',         variable: 'TF_VAR_ci_username'),
                        string(credentialsId: 'ci_password',         variable: 'TF_VAR_ci_password'),
                        string(credentialsId: 'pm_api_endpoint',     variable: 'TF_VAR_pm_api_endpoint'),
                        string(credentialsId: 'pm_api_token',        variable: 'TF_VAR_pm_api_token'),
                        string(credentialsId: 'node_public_key',     variable: 'TF_VAR_node_public_key'),
                        string(credentialsId: 'MacBook_public_key',  variable: 'TF_VAR_MacBook_public_key')
                    ]) {
                        // Environment vars are now available for later stages
                        echo "Terraform credentials set"
                    }
                }
            }
        }
        
        stage('[Terraform] Init & Plan') {
            steps {
                dir('homelab_terraform_configs/create_k3s_cluster') {
                    sh 'terraform init'
                    sh '''
                        terraform plan \
                        -out=tfplan \
                        -var-file=../common/global_variables.tfvars \
                        -var-file=../common/secrets.tfvars
                    '''
                }
            }
        }

        stage('[Terraform] Apply') {
            steps {
                dir('homelab_terraform_configs/create_k3s_cluster') {
                    sh '''
                        terraform apply \
                        -auto-approve tfplan \
                        -var-file=../common/global_variables.tfvars \
                        -var-file=../common/secrets.tfvars
                    '''
                }
            }
        }

        stage('[Ansible] Provisioning') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible_ssh_key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER')]) {
                    dir('homelab_ansible_playbooks') {
                        sh '''
                          chmod 600 "$SSH_KEY_FILE"
                          ansible-playbook k3s_cluster/playbooks/setup_vms.yaml \
                            --private-key "$SSH_KEY_FILE" \
                            -u "$SSH_USER" \
                            --ask-vault-pass
                        '''
                    }
                }
            }
        }
    }
}