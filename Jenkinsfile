#!/usr/bin/env groovy

pipeline {
    agent {label 'Master'}
    tools {
        maven 'Maven_v3.8.6'
    }
    
    environment {
        IMAGE_NAME = 'alexsv1/mavenapp-itschoolproject'
    }

    stages {
        stage('Increment Version') {
            steps {
                script {
                    echo 'Incrementing App Version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_VERSION = "$version-$BUILD_NUMBER"
                }
            }
        }
    
        stage('Build App') {
            steps {
                script {
                    echo "Building the Application..."
                    sh 'mvn clean package'
                }
            }
        }
        stage('Build Image') {
            steps {
                script {
                    echo "Building the Docker Image..."
                    withCredentials([usernamePassword(credentialsId: 'DockerHub-Repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t alexsv1/mavenapp-itschoolproject:${IMAGE_VERSION} ."
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                        sh "docker push alexsv1/mavenapp-itschoolproject:${IMAGE_VERSION}"
                    }
                }
            }
        }
        stage('Provision Server - Terraform') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('Jenkins_AWS_Access_Key_ID')
                AWS_SECRET_ACCESS_KEY = credentials('Jenkins_AWS_Secret_Key')
                TF_VAR_env_prefix = 'test'
            }
            steps {
                script {
                    dir('terraform') {
                        sh "terraform init"
                        sh "terraform apply --auto-approve"
                        EC2_PUBLIC_IP = sh(
                            script: "terraform output ec2_public_ip",
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }



        stage('Deploy to AWS EC2') {
            environment {
                DOCKER_CREDS = credentials('DockerHub-Repo')
            }
            steps {
                script {
                   echo "waiting for EC2 server to initialize" 
                   sleep(time: 90, unit: "SECONDS") 

                   echo 'Deploying Docker Image to EC2...'
                   echo "${EC2_PUBLIC_IP}"

                   def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME}:${IMAGE_VERSION} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
                   def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"

                   sshagent(['server-ssh-key']) {
                       sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
                       sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
                       sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
                   }
                }
            }
        }
    }
}
