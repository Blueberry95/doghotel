pipeline {
    agent any
    parameters {
        choice(name: 'ENVIRONMENTS', choices: ['dev', 'qa', 'prod'], description: 'Environment where the application will be deployed')
    }
    environment {
        FRONTEND_DIR = ""
        FRONTEND_PORT = ""
        BACKEND_PORT = ""
        GIT_BRANCH = ""
        DHP_ENV = "${params.ENVIRONMENTS}"
    }
    stages {
        stage('Clone frontend repo') {
            steps {
                script {
                    if (DHP_ENV == 'dev'){
                        FRONTEND_DIR = '/opt/DogHotel/dev'
                        FRONTEND_PORT = '6001'
                        BACKEND_PORT = '5000'
                        GIT_BRANCH = 'dev_env'
                    } 
                    if (DHP_ENV == 'qa'){
                        FRONTEND_DIR = '/opt/DogHotel/qa'
                        FRONTEND_PORT = '7001'
                        BACKEND_PORT = '6000'
                        GIT_BRANCH = 'ST-335'
                    } 
                    if (DHP_ENV == 'prod'){
                        FRONTEND_DIR = '/opt/DogHotel/app'
                        FRONTEND_PORT = '80'
                        BACKEND_PORT = '5505'
                        GIT_BRANCH = 'dev_frontend'
                    } 
                }  
                cleanWs()
                git branch: "${GIT_BRANCH}", 
                credentialsId: "jenkins-provision-key", 
                url: 'ssh://git@git.epam.com/epm-rdua/epmrduadhp.git'
            }
        }
        stage('build frontend') {
            steps { 
                sh """
                  [ -e package-lock.json ] && rm package-lock.json
                  npm install
                  npm run build
                """
            }
        }
        stage ('create backup FE') {
            steps {
                sh """
                  mkdir -p /tmp/doghotel/backup
                  tar -czf /tmp/doghotel/backup/frontend_${DHP_ENV}_${BUILD_NUMBER}.tar.gz ${FRONTEND_DIR}
                """
            }
        }
        stage ('deploy frontend') {
            steps {
                sh """
                  rm -rf ${FRONTEND_DIR}/*
                  cp -r ./build/* ${FRONTEND_DIR}/
                """
                retry(10) {
                     sh("curl http://doghotel.epm-rdua.projects.epam.com:${FRONTEND_PORT}/ -I --fail")
                }
            }
			post {
				failure('Restore from backup FE') {
					sh """
					rm -rf ${FRONTEND_DIR}/*
					tar xf /tmp/doghotel/backup/frontend_${DHP_ENV}_${BUILD_NUMBER}.tar.gz -C ${FRONTEND_DIR}/
					"""
				}
				success {
					echo "Frontend successfully deployed"
				}
			}
        }

        stage('build backend') {
            steps {
                withMaven(maven: 'Apache Maven 3.6.0') {
                    withCredentials([usernamePassword(credentialsId: 'DogHotel-backend-DB-user', usernameVariable: 'USER', passwordVariable: 'PASSWORD')]) {
                        sh '''
                            mvn -Dflyway.user=$USER -Dflyway.url=jdbc:mariadb://localhost:3306/doghotel -Dflyway.password=$PASSWORD  flyway:migrate
                            mvn clean install
                        '''
                    }
                }
            }
			post {
				failure('Fail') {
					echo "Fail build BE"
            }
			    success {
				    archiveArtifacts artifacts: 'target/*.jar', onlyIfSuccessful: true
					echo "Successfully builded BE"
                }
            }
		}
        stage('deploy backend') {
            steps {
                sh "sudo cp ${WORKSPACE}/target/*.jar /var/lib/jenkins/workspace/test/target/"
                sh "sudo rm -rf /home/SuperAdmin/docker/backend/*.jar && sudo cp ${WORKSPACE}/target/*.jar /home/SuperAdmin/docker/backend/"
                sh "sudo docker build -t backend_${DHP_ENV}:$BUILD_NUMBER /home/SuperAdmin/docker/backend/"
				sh '''
                  if sudo docker container ls --all | grep backend_${DHP_ENV}_latest; then 
                    sudo docker stop backend_${DHP_ENV}_latest
                    sudo docker rename backend_${DHP_ENV}_latest backend_${DHP_ENV}_oldest
                  fi
                '''
                sh "sudo docker run -d -p ${BACKEND_PORT}:5505 --name backend_${DHP_ENV}_latest backend_${DHP_ENV}:$BUILD_NUMBER"     
                sh "sleep 20"
				retry(2) {
					sh('curl http://doghotel.epm-rdua.projects.epam.com:${BACKEND_PORT}/doghotel-api/health -I --fail')
                }
            }
			post {
				failure('Fail') {
					sh '''
                      if sudo docker ps | grep backend_${DHP_ENV}_latest; then
                        sudo docker stop backend_${DHP_ENV}_latest
                        sudo docker rename backend_${DHP_ENV}_oldest backend_${DHP_ENV}_latest
                      fi
                    '''
					sh "sudo docker start backend_${DHP_ENV}_latest"
					echo "Fail deployed BE"
                }
				success {
                    sh '''
                      if sudo docker container ls --all | grep backend_${DHP_ENV}_oldest; then
                        sudo docker rm backend_${DHP_ENV}_oldest
                      fi
                    '''
					echo "Successfully deployed BE"
                }
            }
        }
    }
    post {
        failure('Fail') {
            echo "Fail"
        }
        success {
//            archiveArtifacts artifacts: 'target/*.jar', onlyIfSuccessful: true
            echo "Successfully deployed"
            echo "FRONTEND_PORT: ${FRONTEND_PORT}"
            echo "BACKEND_PORT: ${BACKEND_PORT}"
        }
    }
}
