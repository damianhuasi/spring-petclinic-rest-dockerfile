pipeline {
    agent none

    triggers {
        githubPush()
    }

    stages {

        stage('Build') {
            agent {
                docker {
                    image 'maven:3.9.16-amazoncorretto-21'
                }
            }
            environment {
                MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2"
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
            environment {
                MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2"
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

        stage('SonarQube') {
            agent {
                docker {
                    image 'maven:3.9.16-amazoncorretto-21'
                }
            }
            environment {
                MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2"
                SONAR_USER_HOME = "${WORKSPACE}/.sonar"
            }
            steps {
                checkout scm

                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar -B -ntp'
                }
            }
        }

        stage('Package') {
            agent {
                docker {
                    image 'maven:3.9.16-amazoncorretto-21'
                }
            }
            environment {
                MAVEN_OPTS = "-Dmaven.repo.local=${WORKSPACE}/.m2"
            }
            steps {
                checkout scm

                sh 'mvn clean package -B -ntp -DskipTests'

                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true

                stash includes: 'target/**,Dockerfile,pom.xml', name: 'package'
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
