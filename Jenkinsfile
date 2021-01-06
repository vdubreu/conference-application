pipeline {
    agent any
    tools {
        maven "Maven 3.6.3"
    }
    stages {
        stage('CleanWorkspace') {
          steps {
              cleanWs()
          }
        }
        stage('build conference-app') {
          steps{
              git url: ' https://github.com/promogekko/conference-application.git'
              sh "mvn clean install package"
              archiveArtifacts artifacts: '**/*.war', followSymlinks: false
           }
        }

        stage('Build conference-app Sonarqube') {
          steps {
                withSonarQubeEnv('SonarQube') {
                sh "mvn  clean package sonar:sonar -Dsonar.host_url=$SONAR_HOST_URL "
               }
           }
        }
        stage('Build conference-app Nexus') {
          steps {
                copyArtifacts filter: '**/*.war', fingerprintArtifacts: true, projectName: 'conference-app', selector: lastSuccessful()
                nexusPublisher nexusInstanceId: 'Nexus', nexusRepositoryId: 'maven-releases', packages: [[$class: 'MavenPackage', mavenAssetList: [[classifier: '', extension: '', filePath: 'target/conference-app-3.0.0.war']], mavenCoordinate: [artifactId: 'conference-app', groupId: 'de.codecentric', packaging: 'war', version: '1.0']]]
               }
        }
        stage('Build conference-app Docker image') {
          steps {
                sh "rm -Rf conference-app.war && \
                    wget http://nexus:8081/repository/maven-releases/de/codecentric/conference-app/1.0/conference-app-1.0.war -O ${WORKSPACE}/conference-app.war && \
                    docker build -t conference-app:latest . "
               }
        }
        stage('Run  conference-app Docker ') {
          steps {
                script {
                 def set_container = sh(script: ''' CONTAINER_NAME="conference-app-test"
                                                    OLD="$(docker ps --all --quiet --filter=name="$CONTAINER_NAME")"
                                                    if [ -n "$OLD" ]; then
                                                        docker rm -f $OLD
                                                    fi
                                                    docker run -d --name conference-app-test -p 10090:8080 conference-app
                                         ''')
               }
            }
        }
        stage('AWX - install docker on a remote host') {
          steps {
                ansiColor('xterm') {
                ansibleTower jobTemplate: 'Install_docker', jobType: 'run', throwExceptionWhenFail: false, towerCredentialsId: 'awx', towerLogLevel: 'full', towerServer: 'AWX'
                }
            }
        }
        stage('Push Container to Docker Hub') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'USER_PASSWORD', usernameVariable: 'USER_NAME')]) {
            script {
                 def set_dockerhub = sh(script: ''' docker login -u $USER_NAME -p $USER_PASSWORD
                                                    docker image tag conference-app systemdevformations/conference-app
                                                    docker push systemdevformations/conference-app
                                                 ''')
               }
             }
           }
         }
        stage('Run container conference-app on a remote host') {
          steps {
                ansiColor('xterm') {
                ansibleTower jobTemplate: 'Conference-app', jobType: 'run', throwExceptionWhenFail: false, towerCredentialsId: 'ansible_awx', towerLogLevel: 'full', towerServer: 'AWX'
                }
            }
        }

    }
}
