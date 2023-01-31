@Library("titan-library") _ 

//run the build at 03:10 on every day-of-week from Monday through Friday but only on the main branch
String cron_string = BRANCH_NAME == "main" ? "10 3 * * 1-5" : ""

pipeline {
    agent any
    
    environment {
        NEXUS_VERSION       = "nexus3"
        NEXUS_PROTOCOL      = "https"
        NEXUS_CREDS         = credentials('nexus_user')
        NEXUS_URL           = "${GLOBAL_NEXUS_URL}"
        NEXUS_SNAP_REPO     = "maven-snapshots"
        NEXUS_RELEASE_REPO  = "maven-releases"
        NEXUS_CENTRAL_REPO  = "maven-central"
        NEXUS_GRP_REPO      = "maven-public"
        ARTVERSION          = "${env.BUILD_ID}"

        PACKAGE_TAG         = readMavenPom().getVersion()
        WEBHOOK_URL         = "${GLOBAL_CHATOPS_URL}"
        
        SONAR_AUTH_TOKEN    = credentials('sonarqube_pac_token')
        SONARQUBE_URL       = "${GLOBAL_SONARQUBE_URL}"
        SONAR_HOST_URL      = "${GLOBAL_SONARQUBE_URL}"
        
        BRANCH_NAME         = "${GIT_BRANCH.split("/").size() > 1 ? GIT_BRANCH.split("/")[1] : GIT_BRANCH}"
    }

    triggers {
        cron(cron_string)
    }

    options {
        // Set this to true if you want to clean workspace during the prep stage
        skipDefaultCheckout(false)
        // Console debug options
        timestamps()
        ansiColor('xterm')
    }
        
    stages {
        
        stage('Maven Build') {
            agent {
                docker {
                    image "${GLOBAL_NEXUS_SERVER_URL}/${GLOBAL_NEXUS_REPO_NAME}/java:17.0.2"
                    args '-u root:root'
                }
            }

            steps {
                script{
                    configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) {
                        sh """
                        mvn clean install -s '${MAVEN_SETTINGS}' \
                            --batch-mode \
                            -e \
                            -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn
                        """
                    }
                }
            }

            post {
                always {
                    dir('./') {
                        stash includes: '**/*', name: 'tinkar-origin-test-artifacts'
                    }
                }
            }
        }

        stage('SonarQube Scan') {
            agent {
                docker { 
                    image "${GLOBAL_NEXUS_SERVER_URL}/${GLOBAL_NEXUS_REPO_NAME}/java:17.0.2"
                    args "-u root:root"
                }
            }
            
            steps{
                unstash 'tinkar-origin-test-artifacts'
                withSonarQubeEnv(installationName: 'EKS SonarQube', envOnly: true) {
                    // This expands the evironment variables SONAR_CONFIG_NAME, SONAR_HOST_URL, SONAR_AUTH_TOKEN that can be used by any script.

                    sh """
                        mvn sonar:sonar -Dsonar.login=${SONAR_AUTH_TOKEN} --batch-mode
                    """
                }
            }
               
            post {
                always {
                    echo "post always SonarQube Scan"
                }
            }            
        }
        
        stage("Publish to Nexus Repository Manager") {

            agent {
                docker {
                    image "${GLOBAL_NEXUS_SERVER_URL}/${GLOBAL_NEXUS_REPO_NAME}/java:17.0.2"
                    args '-u root:root'
                }
            }

            steps {

                dir('./') {
                    unstash 'tinkar-origin-test-artifacts'
                }

                script {
                    pomModel = readMavenPom(file: 'pom.xml')                    
                    pomVersion = pomModel.getVersion()
                    isSnapshot = pomVersion.contains("-SNAPSHOT")
                    repositoryId = 'maven-releases'

                    if (isSnapshot) {
                        repositoryId = 'maven-snapshots'
                    } 
                }
             
                configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) { 

                    sh """
                        cat ${MAVEN_SETTINGS}

                        echo "NEXUS_CREDS.USERNAME: ${NEXUS_CREDS_USR}"
                        echo "NEXUS_CREDS.PASS: ${NEXUS_CREDS_PSW}"
                    """

                    sh """
                        mvn deploy \
                        --batch-mode \
                        -e \
                        -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn \
                        -DskipTests \
                        -DskipITs \
                        -Dmaven.main.skip \
                        -Dmaven.test.skip \
                        -s '${MAVEN_SETTINGS}' \
                        -P inject-application-properties \
                        -DrepositoryId='${repositoryId}'
                    """              
                    /*
                    sh """
                        mvn deploy:deploy-file -e -X \
                        -DskipTests -DskipITs \
                        -Dmaven.main.skip \
                        -Dmaven.test.skip \
                        -s '${MAVEN_SETTINGS}' \
                        -P inject-application-properties \
                        -DgroupId 'org.hl7.tinkar-origin-test' \
                        -DartifactId 'loinc-origin' \
                        -Dversion '2.73-SNAPSHOT' \
                        -Dfile 'loinc/target/loinc-origin-2.73-SNAPSHOT.zip' \
                        -Dpackaging 'pom' \
                        -Durl 'https://nexus.build.tinkarbuild.com/' \
                        -DrepositoryId='${repositoryId}'
                    """
                    */
                }

                //     mvn deploy -e -X
            }
        }
    
        /*
        stage('Maven Test') { 
            agent {
                docker { 
                    image "${GLOBAL_NEXUS_SERVER_URL}/${GLOBAL_NEXUS_REPO_NAME}/java:17.0.2"
                    args '-u root:root'
                }
            }

            steps { 
                script{
                    mavenRunTest()
                }
            }
        }
        */
        
        /*
        stage('SonarQube Scan') {
            agent {
                docker { 
                    image "${GLOBAL_NEXUS_SERVER_URL}/${GLOBAL_NEXUS_REPO_NAME}/java:17.0.2"
                    args "-u root:root"
                }
            }
            
            steps{                
                withSonarQubeEnv(installationName: 'EKS SonarQube', envOnly: true) {
                    // This expands the evironment variables SONAR_CONFIG_NAME, SONAR_HOST_URL, SONAR_AUTH_TOKEN that can be used by any script.

                    sh """
                        mvn clean 
                    """
    
                    sh """
                        mvn -e org.sonarsource.scanner.maven:sonar-maven-plugin:3.7.0.1746:sonar -Dsonar.login=${SONAR_AUTH_TOKEN}
                    """
                    
                    script{
                        sonarQubeQualityGate()
                    }

                }
            }
               
            post {
                always {
                    echo "post always SonarQube Scan"
                    
                    dir('loinc/target/') {
                        stash includes: '*.xml', name: 'loinc-sonar'
                    }
                    dir('rxnorm/target/') {
                        stash includes: '*.xml', name: 'rxnorm-sonar'
                    }
                    dir('snomed-us/target/') {
                        stash includes: '*.xml', name: 'snomed-us-sonar'
                    }
                    
                }    
            }            
        }
        */
    
        // stage('push artifacts to nexus') {

            // agent {
            //     docker { 
            //         image "${GLOBAL_NEXUS_SERVER_URL}/${GLOBAL_NEXUS_REPO_NAME}/java:17.0.2"
            //         args "-u root:root"
            //     }
            // }

            // steps {
            //     configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) { 
            //         /*sh "mvn ${MavenConstants.CI_OPTIONS} " 
            //         + "help:active-profiles " 
            //         + "deploy:deploy-file " 
            //         + "-DskipTests -DskipITs " 
            //         + "-Dmaven.main.skip " 
            //         + "-Dmaven.test.skip " 
            //         + "-s '${MAVEN_SETTINGS}' " 
            //         + "-P inject-application-properties "  
            //         + "-Durl=file://${basedir}/loinc/target/*.zip"
            //         */

            //         // mvn help:active-profiles deploy:deploy-file -DskipTests -DskipITs -Dmaven.main.skip -Dmaven.test.skip -s '${MAVEN_SETTINGS}' -P inject-application-properties -Dfile='loinc/target/' -Durl='${GLOBAL_NEXUS_SERVER_URL}'

            //         sh """
            //             mvn deploy
            //         """
            //     }
            // }


            //     /*
            //     configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) { 
            //         sh "mvn ${MavenConstants.CI_OPTIONS} " 
            //         + "help:active-profiles " 
            //         + "deploy " 
            //         + "-DskipTests -DskipITs " 
            //         +  "-Dmaven.main.skip " 
            //         +  "-Dmaven.test.skip " 
            //         +  "-s '${MAVEN_SETTINGS}' " 
            //         +  "-P inject-application-properties "  
            //         //-D settings.security=${MAVEN_SETTINGS_SECURITY}"    
            //     }

                        
// 
            // } */  
        // }
    }


    post {
        always {
            // Clean the workspace after build
            cleanWs(cleanWhenNotBuilt: false,
                deleteDirs: true,
                disableDeferredWipeout: true,
                notFailBuild: true,
                patterns: [
                [pattern: '.gitignore', type: 'INCLUDE']
            ])
        }
    }
}
