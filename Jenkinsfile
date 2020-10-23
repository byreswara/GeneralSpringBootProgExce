pipeline {
    agent any
     tools {
       maven 'maven'
     }
     environment {
        NEXUS_URL = "http://13.212.71.76:8081/content/repositories/releases/"
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
                  sh 'mvn sonar:sonar'
                 }    
             }
         }
	 stage('package') {
            steps {
		tool name: 'maven', type: 'maven'
                sh 'mvn package'
		}
	 }
	 /*stage('Publish Artifacts to Nexus') {
            steps {
              script{
	        sh "mvn -B deploy:deploy-file -Durl=$NEXUS_URL -DrepositoryId=$NEXUS_REPO_ID -DgroupId=$GROUP_ID -Dversion=$NEXUS_VERSION -DartifactId=$ARTIFACT_ID -Dpackaging=war -Dfile=$FILE_NAME"
	      }
            }
	  }
	 */
   stage("Build Docker image and push") {
            steps { 
            sh "docker build -t $ARTIFACT_ID:${BUILD_NUMBER} ."
		        sh "docker tag $ARTIFACT_ID:${BUILD_NUMBER} byresh/$ARTIFACT_ID:${BUILD_NUMBER}"
            withCredentials([usernameColonPassword(credentialsId: 'nexus_cred', variable: 'docker_cred')]) {
            sh "docker push byresh/$ARTIFACT_ID:${BUILD_NUMBER}"
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
	 
        }
}
