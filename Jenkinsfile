node {
  def app
  def dockerfile
  def anchorefile
	
  try {
    stage('Checkout') {
      // Clone the git repository
      checkout scm
      def path = sh returnStdout: true, script: "pwd"
      path = path.trim()
      dockerfile = path + "/Dockerfile"
      anchorefile = path + "/anchore_images"
    }
    stage('OWASP Dependency-Check Vulnerabilities ') {
    dependencyCheck additionalArguments: '''
	    -s "." 
	    -f "ALL"
	    -o "./report/"
	    --prettyPrint
	    --disableYarnAudit''', odcInstallation: 'OWASP Dependency-check'
	    dependencyCheckPublisher pattern: 'report/dependency-check-report.xml'
    }
    stage('SonarQube analysis') {
        def scannerHome = tool 'sonarqube';
        withSonarQubeEnv('sonarserver'){
            sh "${scannerHome}/bin/sonar-scanner \
	      -Dsonar.projectKey=innogrid \
	      -Dsonar.host.url=http://192.168.160.244:9000 \
	      -Dsonar.login=0c50fc8e6a4e1a3a7a5104f0f045367b354fbd0a \
	      -Dsonar.sources=. \
	      -Dsonar.exclusions=report/* \
	      -Dsonar.dependencyCheck.jsonReportPath=./report/dependency-check-report.json \
	      -Dsonar.dependencyCheck.xmlReportPath=./report/dependency-check-report.xml \
	      -Dsonar.dependencyCheck.htmlReportPath=./report/dependency-check-report.html"
        }
    }
    stage('Build Image') {
      // Build the image and push it to a staging repository
      app = docker.build("innogrid/$JOB_NAME", "--network host -f Dockerfile .")
	  docker.withRegistry('https://core.innogrid.duckdns.org', 'harbor') {
	    app.push("$BUILD_NUMBER")
      }
    }
    stage('Anchore Image Scan') {
        writeFile file: anchorefile, text: "core.innogrid.duckdns.org/innogrid" + "/${JOB_NAME}" + ":${BUILD_NUMBER}" + " " + dockerfile
        anchore name: anchorefile, \
	      engineurl: 'http://192.168.160.244:8228/v1', \
	      engineCredentialsId: 'anchore', \
	      annotations: [[key: 'added-by', value: 'jenkins']], \
	      forceAnalyze: true
      }
      currentBuild.result = "SUCCESS"
  } catch (e) {
	echo "Exception=${e}"
        currentBuild.result = 'FAILURE'
	throw e
  } finally {
    stage('Pipeline End') {
      sh script: "echo Pipeline End"
    	}	
     // send slack notification
     if(currentBuild.result.equals("SUCCESS")){
	slackSend (channel: '#jenkins-notification', color: '#00FF00', message: "build success : Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
     }else{
	slackSend (channel: '#jenkins-notification', color: '#F01717', message: "build failed : Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
     }
   }
}
