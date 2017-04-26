podTemplate(label: 'maven-ose', cloud: 'openshift', containers: [
  containerTemplate(name: 'maven', image: "registry.access.redhat.com/openshift3/jenkins-slave-maven-rhel7", ttyEnabled: true, command: 'cat', workingDir: '/home/jenkins'),
],
volumes: [configMapVolume(configMapName: 'jenkins-maven-settings', mountPath: '/etc/maven'),
          persistentVolumeClaim(claimName: 'maven-local-repo', mountPath: '/etc/.m2repo')]) {

def appName = "ose-api-app"
def devProject = "api-app-dev"
//def uatProject = "api-app-uat"
//def prodProject = "api-app-prod"
def version

node('maven-ose') {
        container(name: 'maven', cloud: 'openshift') {


    	def WORKSPACE = pwd()
    	//def mvnHome = tool 'maven'
    	env.KUBECONFIG = pwd() + "/.kubeconfig"

   	stage 'Checkout'

       	   	checkout scm

   	stage 'Maven Build'

             	try {
            		sh """
           		set -x
            		mvn build-helper:parse-version versions:set -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.incrementalVersion}-\${BUILD_NUMBER}
            		mvn clean install
            		"""

            		step([$class: 'ArtifactArchiver', artifacts: '**/target/*.war, **/target/*.jar', fingerprint: true])
       	     	} catch(e) {
                	currentBuild.result = 'FAILURE'
                	throw e
             	} finally {
            		processStageResult()
        	}

      	stage "OpenShift Dev Build"

        	version = parseVersion("${WORKSPACE}/pom.xml")

        	login()

        	sh """
       		set +x

         	currentOutputName=\$(oc get bc ${appName} -n ${devProject} --template='{{ .spec.output.to.name }}')

         	newImageName=\${currentOutputName%:*}:${version}

         	oc patch bc ${appName} -n ${devProject} -p "{ \\"spec\\": { \\"output\\": { \\"to\\": { \\"name\\": \\"\${newImageName}\\" } } } }"

         	mkdir -p ${WORKSPACE}//target/s2i-build/deployments
         	cp ${WORKSPACE}//target/*.jar ${WORKSPACE}//target/s2i-build/deployments/
         	oc start-build ${appName} -n ${devProject} --follow=true --wait=true --from-dir="${WORKSPACE}//target/s2i-build"

       		"""

      	stage "Dev Deployment"

        	login()

        	deployApp(appName, devProject, version)

        	validateDeployment(appName,devProject)
	
	}
    }
}

def processStageResult() {

    if (currentBuild.result != null) {
        sh "exit 1"
    }
}

def login() {
    sh """
       set +x
       oc login --certificate-authority=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt --token=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) https://kubernetes.default.svc.cluster.local >/dev/null 2>&1 || echo 'OpenShift login failed'
       """
}

def parseVersion(String filename) {
  def matcher = readFile(filename) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}

def deployApp(appName, namespace, version) {
            sh """
          set +x

          newDeploymentImageName=${appName}:${version}

          imageReference=\$(oc get is ${appName} -n ${namespace} -o jsonpath="{.status.tags[?(@.tag==\\"${version}\\")].items[*].dockerImageReference}")

          oc patch dc/${appName} -n ${namespace} -p "{\\"spec\\":{\\"template\\":{\\"spec\\":{\\"containers\\":[{\\"name\\":\\"${appName}\\",\\"image\\": \\"\${imageReference}\\" } ]}}, \\"triggers\\": [ { \\"type\\": \\"ImageChange\\", \\"imageChangeParams\\": { \\"containerNames\\": [ \\"${appName}\\" ], \\"from\\": { \\"kind\\": \\"ImageStreamTag\\", \\"name\\": \\"\${newDeploymentImageName}\\" } } } ] }}"

          oc deploy ${appName} -n ${namespace} --latest

          # Sleep for a few moments
          sleep 5
        """


}

def acceptanceCheck(String appName, String namespace) {

    sh """
      set +x

      COUNTER=0
      DELAY=5
      MAX_COUNTER=30

      echo "Running Acceptance Check of ${appName} in project ${namespace}"

     set +e

      while [ \$COUNTER -lt \$MAX_COUNTER ]
      do

        RESPONSE=\$(curl -s -o /dev/null -w '%{http_code}\\n' http://${appName}.${namespace}.svc.cluster.local:8080/rest/api/pods)

        if [ \$RESPONSE -eq 200 ]; then
            echo
            echo "Application Verified"
            break
        fi

        if [ \$COUNTER -eq \$MAX_COUNTER ]; then
          echo "Max Validation Attempts Exceeded. Failed Verifying Application Deployment..."
          exit 1
        fi

        sleep \$DELAY

      done

      set -e
      """

}
