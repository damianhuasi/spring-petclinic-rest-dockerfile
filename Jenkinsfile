pipeline {
    agent none

    environment {
        MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2"
        SONAR_USER_HOME = "${WORKSPACE}/.sonar"
    }

    stages {
 
        stage('CI') {

            agent {
                docker {
                    image 'maven:3.9.16-amazoncorretto-21'
                }
            }

            stages {

                stage('Build') {
                    steps {
                        checkout scm
                        sh 'mvn clean compile -B -ntp'
                    }
                }

                stage('Testing (JUnit + JaCoCo)') {
                    steps {
                        sh 'mvn test jacoco:report -B -ntp'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }

                stage('SonarQube') {
                    steps {
                        withSonarQubeEnv('sonarqube') {
                            sh 'mvn sonar:sonar -B -ntp'
                        }
                    }
                }

                stage('Package') {
                    steps {
                        sh 'mvn package -DskipTests -B -ntp'
                        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                    }
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

            steps {

                checkout scm

                unstash 'package'

                sh 'docker --version'
                sh 'docker images'

                script {

                    def pom = readMavenPom file: 'pom.xml'

                    def image = "eloydamian/${pom.artifactId}"

                    echo "Imagen: ${image}"
                    echo "Version: ${pom.version}"

                    sh """
                        docker build \
                        -t ${image}:${pom.version} \
                        -t ${image}:latest .
                    """

                    sh """
                        echo "${DOCKERHUB_CREDENTIALS_PSW}" | \
                        docker login \
                        -u "${DOCKERHUB_CREDENTIALS_USR}" \
                        --password-stdin
                    """

                    sh "docker push ${image}:${pom.version}"
                    sh "docker push ${image}:latest"

                    sh "docker logout"
                }
            }
        }
    }

    // post {
    //     always {
    //         cleanWs()
    //     }
    // }
}
