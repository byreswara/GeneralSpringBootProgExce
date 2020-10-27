pipeline {
    agent any
     tools {
       maven 'maven'
     }
     environment {
        NEXUS_URL = "http://13.212.163.49:8081/repository/releases/"
	NEXUS_REPO_ID = "releases"
	GROUP_ID="`echo -e 'setns x=http://maven.apache.org/POM/4.0.0\ncat /x:project/x:groupId/text()' | xmllint --shell pom.xml | grep -v /`"
        ARTIFACT_ID="`echo -e 'setns x=http://maven.apache.org/POM/4.0.0\ncat /x:project/x:artifactId/text()' | xmllint --shell pom.xml | grep -v /`"
      	VERSION="`echo -e 'setns x=http://maven.apache.org/POM/4.0.0\ncat /x:project/x:version/text()' | xmllint --shell pom.xml | grep -v /`"
	NEXUS_VERSION="`echo -e 'setns x=http://maven.apache.org/POM/4.0.0\ncat /x:project/x:version/text()' | xmllint --shell pom.xml | grep -v /`.${BUILD_NUMBER}"
        FILE_NAME="target/${ARTIFACT_ID}-${VERSION}.jar"
      }
      stages {
	stage('Unit Test') {
            steps {
		tool name: 'maven', type: 'maven'
                sh 'mvn clean test'
	    }
	 }
	 stage("SonarQube analysis") {
             steps {
                 withSonarQubeEnv('sonar') {
                  //sh 'mvn sonar:sonar'
		    sh 'mvn sonar:sonar -Dsonar.projectKey=myproject  -Dsonar.host.url=http://13.212.163.49:9000  -Dsonar.login=6ae04962f89ae661db1d9e291e011a4526a859c8'
                 }    
             }
         }
	/* stage("Quality Gate status") {
	  steps {
	    script {
                def qualitygate = waitForQualityGate()
                if (qualitygate.status != "OK") {
                  error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"	
		}
	    }
	    }
	}*/
	 stage('package') {
            steps {
		tool name: 'maven', type: 'maven'
                sh 'mvn package'
		}
	 }
	stage('Publish Artifacts to Nexus') {
            steps {
              script{
	        sh "mvn -B deploy:deploy-file -Durl=$NEXUS_URL -DrepositoryId=$NEXUS_REPO_ID -DgroupId=$GROUP_ID -Dversion=$NEXUS_VERSION -DartifactId=$ARTIFACT_ID -Dpackaging=war -Dfile=$FILE_NAME"
	      }
            }
	  }
	
   /*stage("Build Docker image and push to dockerhub") {
            steps { 
            sh "docker build -t myapp:v0.${BUILD_NUMBER} ."
            sh "docker tag myapp:v0.${BUILD_NUMBER} byresh/myapp:v0.${BUILD_NUMBER}"
	    withCredentials([usernamePassword(credentialsId: 'docker_cred', passwordVariable: 'pass', usernameVariable: 'user')]) {
            sh "docker login --username $user --password $pass"
	    sh "docker push byresh/myapp:v0.${BUILD_NUMBER}"
            }
           }
	  }*/
     stage("Build Docker image and push to docker hosted nexus repo") {
            steps { 
            sh "docker build -t myapp:v0.${BUILD_NUMBER} ."
            sh "docker tag myapp:v0.${BUILD_NUMBER} 13.212.163.49:8083/myapp:v0.${BUILD_NUMBER}"
	    withDockerRegistry(credentialsId: 'dockrrigistry', url: 'http://13.212.163.49:8083') {
             sh "docker push 13.212.163.49:8083/myapp:v0.${BUILD_NUMBER}"
            }
           }
	  }  
	 stage('Waiting for Approval'){
           steps {
	      script {
	        slackSend channel: '#anz',
                          message: "Need Approval to deploy: JobName: ${env.JOB_NAME} BuildNumber: ${env.BUILD_NUMBER} and (<${env.BUILD_URL}input|Click here to approve>)",
	                  color: 'good'
                    try {
                        timeout(time:30, unit:'MINUTES') {
                            env.APPROVE_SIT = input message: 'push docker image', ok: 'Continue',
                                parameters: [choice(name: 'push docker image', choices: 'YES\nNO', description: 'push docker image?')]
                            if (env.APPROVE_SIT == 'YES')
				{
                                env.DSIT = true
                                } 
			    else
			       {
                                env.DSIT = false
                                }
                        }
                    } catch (error) {
                        env.DSIT = true
                        echo 'Timeout has been reached! Deploy to SIT automatically activated'
                    }
                }
	   }
	 }
	 stage("Deploy to SIT Environment") {
            steps {
		sshPublisher(publishers: [sshPublisherDesc(configName: 'ansible-server', 
							   transfers: [sshTransfer(cleanRemote: false, 
							   excludes: '', execCommand: 'docker run -itd 13.212.163.49:8083/myapp:v0.${BUILD_NUMBER}', 
							   flatten: false, 
							   makeEmptyDirs: false, 
							  noDefaultExcludes: false, 
							  patternSeparator: '[, ]+', 
							  remoteDirectory: '', 
							  remoteDirectorySDF: false, 
							  removePrefix: '', 
			          			  sourceFiles: '')], 
                             				  usePromotionTimestamp: false, 
							  useWorkspaceInPromotion: false, 
							  verbose: false)])
	    }
	  }
        }
}
