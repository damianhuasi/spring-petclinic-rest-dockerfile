pipeline {
    agent {
        docker {
            //image 'maven:3.9.15-eclipse-temurin-21'
             //image 'maven:3.8.8-eclipse-temurin-21'
             image 'maven:3.9.16-amazoncorretto-21'
        }
    }    
    //Se usa el de contenedor Docker con Maven preinstalado.
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
        ARTIFACTORY_CREDENTIALS = credentials('artifactory-credentials')
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
        
        // stage("Quality Gate"){
        //     steps{
        //         timeout(time: 5, unit: 'MINUTES') {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests -B -ntp'
            }
        }       
             
        stage('Artifactory') {
            steps {
                // script {

                //     sh 'env | sort'
                //     sh 'mvn --version'
                //     env.MAVEN_HOME = '/usr/share/maven'

                //     def releases = 'spring-petclinic-rest-release'
                //     def snapshots = 'spring-petclinic-rest-snapshot'

                //     def server = Artifactory.server 'artifactory'

                //     def rtMaven = Artifactory.newMavenBuild()
                //     rtMaven.deployer server: server, releaseRepo: releases, snapshotRepo: snapshots

                //     rtMaven.deployer
                //         .addProperty('build.url', env.RUN_DISPLAY_URL)
                //         .addProperty('build.user', env.USER)

                //     def buildInfo = rtMaven.run pom: 'pom.xml', goals: 'clean install -B -ntp -DskipTests'
                //     server.publishBuildInfo buildInfo                    
                // }
        
                withCredentials([file(credentialsId: 'artifactory-settings', variable: 'M2_SETTINGS')]) {
                    sh 'env | sort'
                    sh 'mvn clean package -B -DskipTests -s ${M2_SETTINGS}'
                }                

                script{
                    def releaseRepo = 'spring-petclinic-rest-release'
                    def snapshotRepo = 'spring-petclinic-rest-snapshot'
                    
                    def server = Artifactory.server 'artifactory'
                    
                    def pom = readMavenPom file: 'pom.xml'
                    println pom.groupId

                    def groupIdPath = pom.groupId.replaceAll("\\.", "/")
                    println groupIdPath

                    def uploadSpec = """
                        {
                            "files": [
                                {
                                    "pattern": "target/.*.jar",
                                    "target": "${releaseRepo}/${groupIdPath}/${pom.artifactId}/${pom.version}/",
                                    "regexp": "true",
                                    "props": "build.url=${RUN_DISPLAY_URL};build.user=${USER}"
                                }
                            ]
                        }
                    """
                    server.upload spec: uploadSpec
                }
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
