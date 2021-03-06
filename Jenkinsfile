def mvnHome
def remote = [:]
        remote.name = 'deploy'
        remote.host = '192.168.33.20'
        remote.user = 'root'
        remote.password = 'vagrant'
        remote.allowAnyHosts = true
pipeline {
    
    agent none
    
    stages {
        //def mvnHome
        stage ('Preparation') {
            agent {
                label 'slave'
            }
            steps {
                git 'https://github.com/RanadevKranthi/Maven-Java-Project.git'
                stash 'Source'
                script{
                    mvnHome = tool 'maven3.6'
                }
            }
        }
        stage ('Static Analysis'){
            agent {
                label "slave"
            }
            steps {
                sh "'${mvnHome}/bin/mvn' clean cobertura:cobertura"         
            }
            post {
                success {
                    cobertura autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: 'target/site/cobertura/coverage.xml', conditionalCoverageTargets: '70, 0, 0', failUnhealthy: false, failUnstable: false, lineCoverageTargets: '80, 0, 0', maxNumberOfBuilds: 0, methodCoverageTargets: '80, 0, 0', onlyStable: false, sourceEncoding: 'ASCII', zoomCoverageChart: false
                }
            }
        }
        stage ('build'){
            agent {
                label "slave"
            }
            steps {
                sh "'${mvnHome}/bin/mvn' clean package"         
            }
            post {
                success {
                    junit 'target/surefire-reports/*.xml'
                    archiveArtifacts '**/*.war'
                    fingerprint '**/*.war'
                }
            }
        }
        stage('Deploy-to-Stage') {
             agent {
                label 'slave'
            }
            //SSH-Steps-Plugin should be installed
            //SCP-Publisher Plugin (Optional)
            steps {
                //sshScript remote: remote, script: "abc.sh"    
            sshPut remote: remote, from: 'target/java-maven-1.0-SNAPSHOT.war', into: '/root/staging-server/webapps'             
            }
        }
        stage ('Integration-Test') {
            agent {
                label "slave"
            }
            steps {
                parallel (
                    'integration': { 
                        unstash 'Source'
                        sh "'${mvnHome}/bin/mvn' clean verify"
                                                        
                    }, 'quality': {
                        unstash 'Source'
                        sh "'${mvnHome}/bin/mvn' clean test"
                    }
                )
            }
        }
        stage ('approve') {
            agent {
                label "slave"
            }
            steps {
                timeout(time: 7, unit: 'DAYS') {
                    input message: 'Do you want to deploy?', submitter: 'admin'
                }
            }
        }
        stage ('Deploy-to-staging') {
            agent {
                label "slave"
            }
            steps {
                unstash 'Source'
                sh "'${mvnHome}/bin/mvn' clean deploy"              
            }
            post {
                success {
                    archiveArtifacts '**/*.war'
                }
            }
            }

            stage ('Deploy-to-ansible') {
            agent {
                label "slave"
            }
            steps{
                sh label: '', script: '''cd target
                                         rm -rf webapp.war
                                         mv *.war webapp.war'''
                }

            post {
                success {
                    sshPublisher(publishers: [sshPublisherDesc(configName: 'ansiblemaster', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '''cd ~/ansible-files
                git pull origin master
                cd ansibleRoles
                ansible-playbook tomcat.yml''', execTimeout: 600000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: 'target', sourceFiles: '**/*.war')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
                }
            }

        }
    }
}   
