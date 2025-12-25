pipeline {
    agent any

    parameters {
        string(name: 'DOCKER_TAG', defaultValue: 'latest', description: 'Docker tag')
    }
    
    tools {
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/AdarshJain-dev/Multi-Tier-BankApp-CI.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }
        stage('FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=bankapp -Dsonar.projectKey=bankapp \
                            -Dsonar.java.binaries=target '''
                }
            }
        }
        stage('Building and publish Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings-devopsshack', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }
        stage('Docker build & tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t adarshjain428/bankapp:${params.DOCKER_TAG} ."
                    }
                }
            }
        }
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o dimage.html adarshjain428/bankapp:${params.DOCKER_TAG}"
            }
        }
        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push adarshjain428/bankapp:${params.DOCKER_TAG}"
                    }
                }
            }
        }
        stage('Update image tag in github') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'git-cred', gitToolName: 'Default')]) {
                    sh ''' git clone https://github.com/AdarshJain-dev/Multi-Tier-BankApp-CD.git
                           cd Multi-Tier-BankApp-CD
                           ls -l bankapp
                           repo_dir=$(pwd)
                           
                           sed -i 's|image: adarshjain428/bankapp:.*|image: adarshjain428/bankapp:'${DOCKER_TAG}'|' ${repo_dir}/bankapp/bankapp-ds.yml
                       '''
                    sh ''' echo "Updated YAML file contents:"
                           cat Multi-Tier-BankApp-CD/bankapp/bankapp-ds.yml
                       '''
                       
                    sh '''
                          cd Multi-Tier-BankApp-CD
                          git config user.email "anany.0304pandey@gmail.com"
                          git config user.name "Jenkins CI"
                       '''
                    sh '''
                          cd Multi-Tier-BankApp-CD
                          ls
                          git add bankapp/bankapp-ds.yml
                          git commit -m "Update image tag to ${DOCKER_TAG}"
                          git push origin main
                       '''
                }
            }
        }
    }
}
