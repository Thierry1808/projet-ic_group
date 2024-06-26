@Library("Library")_

pipeline {
  environment {
        IMAGE_NAME = "ic-webapp"
        APP_CONTAINER_PORT = "8080"
        DOCKERHUB_ID = "thierry237"
        DOCKERHUB_PASSWORD = credentials('dockerhub_password')
        ANSIBLE_IMAGE_AGENT = "registry.gitlab.com/robconnolly/docker-ansible:latest"
    }
    agent none
    stages {
        stage ( 'Build image') {
            agent any
            steps {
                script {
                    sh 'docker build --no-cache -f ./sources/app/${DOCKERFILE_NAME} -t ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG ./sources/app'
                }
            }
        }
        stage ('Run container based on builded image') {
            agent any
            steps {
                script {
                    sh '''
                        echo "Cleaning existing container if exist"
                        docker ps -a | grep -i $IMAGE_NAME && docker rm -f ${IMAGE_NAME}
                        docker run --name ${IMAGE_NAME} -d -p $APP_EXPOSED_PORT:$APP_CONTAINER_PORT ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
                        sleep 5
                    '''
                }
            }
        }
        stage('Test Image') {
            agent any
            steps {
                script {
                    sh '''
                        curl -I http://${HOST_IP}:${APP_EXPOSED_PORT} | grep -i "200"
                    '''
                }
            }
        }
        stage('Clean container') {
            agent any
            steps {
                script {
                    sh '''
                    docker stop $IMAGE_NAME
                    docker rm $IMAGE_NAME
                '''
                }
            }
        }
        stage('Login and Push Image on docker hub') {
            agent any
            steps {
                script {
                    sh '''
                        echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_ID --password-stdin
                        docker push ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG 
                    '''
                }
            }
        }
        stage ('Ansible environment') {
            agent any
            environment {
                VAULT_KEY = credentials ('vault_key')
                PRIVATE_KEY = credentials ('private_key')
            }
            steps {
                script {
                    sh '''
                        echo $VAULT_KEY > vault.key
                        echo $PRIVATE_KEY > id_rsa
                        chmod 600 id_rsa
                    '''
                }
            }
        }
        stage ('Deploy application') {
            agent {  docker  { image 'registry.gitlab.com/robconnolly/docker-ansible:latest' }  }
            stages {
                stage ("Install Ansible role dependencies") {
                    steps {
                        script {
                            sh 'echo launch ansible-galaxy install -r roles/requirement.yml if needed'
                        }
                    }
                }
                stage ("Ping target hosts") {
                    steps {
                        script {
                            sh '''
                                apt update -y
                                apt install sshpass -y
                                export ANSIBLE_CONFIG=$(pwd)/sources/ansible-ressources/ansible.cfg
                                ansible all -m ping --private-key id_rsa -i $(pwd)/sources/ansible-ressources/hosts.yml -l prod
                            '''
                        }
                    }
                }
                stage ("Check all playbook syntax") {
                    steps {
                        script {
                            sh '''
                                export ANSIBLE_CONFIG=$(pwd)/sources/ansible-ressources/ansible.cfg
                                ansible-lint -x 306 sources/ansible-ressources/playbook/* || echo passing linter
                                echo ${GIT_BRANCH}
                            '''
                        }
                    }
                }
                stage ("Deploy in prod") {
                    when { expression { GIT_BRANCH == 'origin/main' } }
                    stages {
                        stage ("PROD - Install Docker on all hosts") {
                            steps {
                                script {
                                    sh '''
                                        export ANSIBLE_CONFIG=$(pwd)/sources/ansible-ressources/ansible.cfg
                                        ansible-playbook sources/ansible-ressources/playbook/install-docker.yml --vault-password-file vault.key --private-key id_rsa -l odoo_server,pg_admin_server
                                    '''
                                }
                            }
                        }
                        stage ("PROD - Deploy pgadmin") {
                            steps {
                                script {
                                    sh '''
                                        export ANSIBLE_CONFIG=$(pwd)/sources/ansible-ressources/ansible.cfg
                                        ansible-playbook sources/ansible-ressources/playbook/deploy-pgadmin.yml --vault-password-file vault.key --private-key id_rsa -l pg_admin
                                    '''
                                }
                            }
                        }
                        stage ("PROD - Deploy odoo") {
                            steps {
                                script {
                                    sh '''
                                        export ANSIBLE_CONFIG=$(pwd)/sources/ansible-ressources/ansible.cfg
                                        ansible-playbook sources/ansible-ressources/playbook/deploy-odoo.yml --vault-password-file vault.key --private-key id_rsa -l odoo
                                    '''
                                }
                            }
                        }
                        stage ("PROD - Deploy ic-webapp") {
                            steps {
                                script {
                                    sh '''
                                        export ANSIBLE_CONFIG=$(pwd)/sources/ansible-ressources/ansible.cfg
                                        ansible-playbook sources/ansible-ressources/playbooks/deploy-ic-webapp.yml --vault-password-file vault.key --private-key id_rsa -l ic_webapp
                                    '''
                                }
                            }
                        }
                    }
                }
                    
            }
        }
    }
    post {
        always {
            script {
                slackNotifier currentBuild.result
            }
        }
    }  
}    
