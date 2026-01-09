pipeline {
    agent any

    parameters {
        string(name: 'DOCKER_TAG', defaultValue: 'latest', description: 'Docker tag')
    }
    
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    
    // environment {
    //     SCANNER_HOME = tool 'sonar-scanner'
    // }

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
        // stage('SonarQube Analysis') {
        //     steps {
        //         withSonarQubeEnv('sonar') {
        //             sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=bankapp -Dsonar.projectKey=bankapp \
        //                     -Dsonar.java.binaries=target '''
        //         }
        //     }
        // }
        stage('Building and publish Nexus') {
            steps {
                // withMaven(globalMavenSettingsConfig: 'maven-settings-devopsshack', maven: 'maven3', traceability: true) {
                    sh 'mvn clean package -DskipTests=true'
                // }
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
        stage('Update image tag in github (Blue-Green safe)') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'git-cred', gitToolName: 'Default')]) {
                    sh '''
                        set -e
        
                        git clone https://github.com/AdarshJain-dev/Multi-Tier-BankApp-CD.git
                        cd Multi-Tier-BankApp-CD
        
                        VALUES_FILE="bankapp-helm/values.yaml"
        
                        echo "Using values file: $VALUES_FILE"
                        cat $VALUES_FILE
        
                        ACTIVE_COLOR=$(grep '^activeColor:' $VALUES_FILE | awk '{print $2}')
                        echo "Active color is: $ACTIVE_COLOR"
        
                        if [ "$ACTIVE_COLOR" = "blue" ]; then
                            echo "Blue is LIVE → updating GREEN image tag"
                            sed -i "s/^  greenTag:.*/  greenTag: ${DOCKER_TAG}/" $VALUES_FILE
                        else
                            echo "Green is LIVE → updating BLUE image tag"
                            sed -i "s/^  blueTag:.*/  blueTag: ${DOCKER_TAG}/" $VALUES_FILE
                        fi
        
                        echo "Updated values.yaml:"
                        cat $VALUES_FILE
        
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins CI"
        
                        git add $VALUES_FILE
                        git commit -m "Update inactive color image tag to ${DOCKER_TAG}"
                        git push origin main
                    '''
                }
            }
        }

    }
}
