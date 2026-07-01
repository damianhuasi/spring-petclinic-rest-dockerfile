pipeline {

    agent none

    environment {
        MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2"
        SONAR_USER_HOME = "${WORKSPACE}/.sonar"
    }

    stages {

        stage('Build') {
            agent {
                docker {
                    image 'maven:3.9.16-amazoncorretto-21'
                }
            }
            steps {
                checkout scm
                sh 'mvn clean compile -B -ntp'
            }
        }

        stage('Testing (JUnit + JaCoCo)') {
            agent {
                docker {
                    image 'maven:3.9.16-amazoncorretto-21'
                }
            }
            steps {
                checkout scm
                sh 'mvn test jacoco:report -B -ntp'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Sonar') {
            agent {
                docker {
                    image 'maven:3.9.16-amazoncorretto-21'
                }
            }
            steps {
                checkout scm
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar -B -ntp'
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
                DOCKERHUB = credentials('dockerhub-credentials')
            }
            options { skipDefaultCheckout() }

            steps { 
 
                sh 'mvn clean package -DskipTests -B -ntp'

                script {

                    def pom = readMavenPom file: 'pom.xml'
                    def image = "eloydamian/${pom.artifactId}"

                    // Forma 1: Usando comandos Docker directamente

                    sh 'docker build --help'
                    sh "docker build -t ${image}:${pom.version} . -t ${image}:latest"
                    sh 'docker images'

                    sh 'echo "$DOCKERHUB_CREDENTIALS_PSW" | docker login -u "$DOCKERHUB_CREDENTIALS_USR" --password-stdin'
                    sh "docker push ${image}:${pom.version}"
                    sh "docker push ${image}:latest"
                    sh 'docker logout'
                }
            }

            // post {
            //     always {
            //         cleanWs()
            //     }
            // }
        }
    }
}
