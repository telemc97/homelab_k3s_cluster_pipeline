pipeline {
    agent { label 'control_node_agent0' }

    parameters {
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: true, description: 'Clean the workspace before build')
    }

    environment {
        TF_VAR_ci_username             = ''
        TF_VAR_ci_password             = ''
        TF_VAR_pm_api_endpoint         = ''
        TF_VAR_pm_api_token            = ''
        TF_VAR_ssh_ansible_public_key  = ''
        TF_VAR_ssh_auxilery_public_key = ''
    }

    stages {

        stage('Pre-clean') {
            when {
                expression { params.CLEAN_WORKSPACE }
            }
            steps {
                echo 'Cleaning workspace...'
                deleteDir()
            }
        }

        stage('Clean and Init Submodules') {
            steps {
                script {
                    echo 'Cleaning any stale submodules and reinitializing...'
                    sh '''
                        git submodule deinit -f .
                        rm -rf .git/modules/*
                        git submodule update --init --recursive --force
                    '''
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

        stage('Setup Terraform Env') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'ci_username', variable: 'TF_VAR_ci_username'),
                        string(credentialsId: 'ci_password', variable: 'TF_VAR_ci_password'),
                        string(credentialsId: 'pm_api_endpoint', variable: 'TF_VAR_pm_api_endpoint'),
                        string(credentialsId: 'pm_api_token', variable: 'TF_VAR_pm_api_token'),
                        string(credentialsId: 'ssh_auxilery_public_key', variable: 'TF_VAR_ssh_auxilery_public_key')
                    ]) {
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
                    '''
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
