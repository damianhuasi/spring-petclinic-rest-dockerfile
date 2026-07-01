pipeline {
    agent {
        docker { 
             image 'maven:3.9.16-amazoncorretto-21'
        }
    }    
    //Se usa el de contenedor Docker 
    // tools {
    //     maven 'maven3.9.16'
    // }
    triggers {
        // cron('* * * * *')
        githubPush()
    }

    environment {
        MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2"
        SONAR_USER_HOME = "${WORKSPACE}/.sonar" 
    }

    stages { 
        stage('Build') {
            steps {
                sh 'mvn clean compile -B -ntp'
            }
        }
        stage('Testing (JUnit + JaCoCo)') {
            steps {
                sh 'mvn test jacoco:report -B -ntp'
            }
            post { 
                success {
                    junit 'target/surefire-reports/*.xml' 
                }
            }
        }

        stage('Sonarqube') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'env | sort'
                    sh 'mvn sonar:sonar -B -ntp'
                }
            }
        } 

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests -B -ntp'
            }
        }       
              
    }
     stage('DockerHub') {
            agent {
                docker {
                    image 'docker:29.4.0-cli'
                    args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock' 
                }
            }
            environment {
                DOCKER_CONFIG = "${WORKSPACE}/.docker"
                DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
            }            
            options { skipDefaultCheckout() }
            steps {
                // sh 'ls -la'
                sh 'docker --version'
                sh 'docker images'

                script {

                    def pom = readMavenPom file: 'pom.xml'
                    def image = "eloydamian/${pom.artifactId}"
 

                    sh 'docker build --help'
                    sh "docker build -t ${image}:${pom.version} . -t ${image}:latest"
                    sh 'docker images'

                    sh 'echo "$DOCKERHUB_CREDENTIALS_PSW" | docker login -u "$DOCKERHUB_CREDENTIALS_USR" --password-stdin'
                    sh "docker push ${image}:${pom.version}"
                    sh "docker push ${image}:latest"
                    sh 'docker logout' 
                }
            }
        } 

    post { 
        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
        cleanup {
            cleanWs()
        }
    }
}
