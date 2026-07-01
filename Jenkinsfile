pipeline {
    agent {
        docker {
            image 'maven:3.9.15-eclipse-temurin-21'
        }
    }

    triggers {
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
                sh 'ls -lh target'
            }
        }

        stage('DockerHub') {

            agent {
                docker {
                    image 'docker:28-cli'
                    args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }

            environment {
                DOCKER_CONFIG = "${WORKSPACE}/.docker"
                DOCKERHUB = credentials('dockerhub-credentials')
            }

            steps {

                checkout scm

                sh 'docker version'
                sh 'docker info'

                sh 'ls -la'
                sh 'ls -la target'

                script {

                    def pom = readMavenPom file: 'pom.xml'

                    def image = "eloydamian/${pom.artifactId}"

                    echo "Imagen: ${image}"
                    echo "Versión: ${pom.version}"

                    sh """
                        docker build \
                        -t ${image}:${pom.version} \
                        -t ${image}:latest .
                    """

                    sh """
                        echo "${DOCKERHUB_PSW}" | docker login \
                        -u "${DOCKERHUB_USR}" \
                        --password-stdin
                    """

                    sh "docker push ${image}:${pom.version}"
                    sh "docker push ${image}:latest"

                    sh "docker logout"
                }
            }
        }
    }

    post {

        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }

        always {
            cleanWs()
        }
    }
}
