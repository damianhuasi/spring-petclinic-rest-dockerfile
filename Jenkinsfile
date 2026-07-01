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
    post { 
        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
        cleanup {
            cleanWs()
        }
    }
}
