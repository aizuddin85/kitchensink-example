// Jenkinsfile for MLBParks
podTemplate(
  label: "skopeo-pod",
  cloud: "openshift",
  inheritFrom: "maven",
  containers: [
    containerTemplate(
      name: "jnlp",
      image: "docker-registry.default.svc:5000/jenkins/jenkins-slave-appdev",
      resourceRequestMemory: "1Gi",
      resourceLimitMemory: "2Gi"
    )
  ]
) {
  node('skopeo-pod') {
    // Your Pipeline Code goes here. Make sure to use the ${GUID} and ${CLUSTER} parameters where appropriate
    // You need to build the application in directory `MLBParks`.
    // Also copy "../nexus_settings.xml" to your build directory
    // and replace 'GUID' in the file with your ${GUID} to point to >your< Nexus instance
    def appNamespace = "kitchensink"
    def sonarqubeUrl = "http://sonarqube-sonarqube.cloudapps.bytewise.com.my/"
    def nexusRelUrl = "http://nexus3.nexus3.svc.cluster.local:8081/repository/releases"
    def mavenMirrorUrl = "http://nexus3-nexus3.cloudapps.bytewise.com.my/repository/maven-all-public"
    // Defining base Maven command.
    def mvnCmd = "mvn -s ../nexus_settings.xml"

    // Checking out source.
    stage("Checking out source code") {
      git branch: 'master', changelog: false, poll: false, url: 'https://github.com/aizuddin85/kitchensink-example.git'
    }

    // For those stages below use this directory as contextDir and set build definition.
    def groupId    = getGroupIdFromPom("pom.xml")
    def artifactId = getArtifactIdFromPom("pom.xml")
    def version    = getVersionFromPom("pom.xml")
    def devTag  = "${version}-${BUILD_NUMBER}"
    def prodTag = "${version}"
    def destApp   = "kitchensink"
    def appName = "${destApp}"
    def destSvc = "kitchensink-blue"
    def activeApp = ""

    stage("Building Target"){
      // Clean packaging of source code and skip maven test.
      sh "${mvnCmd} clean package -DskipTests=true"
    }

    stage("Performing Unit Test"){
      // Now run arquillian test.
      sh "${mvnCmd} test"
    }

    stage("Running SonarQube code analysis"){
      sh "${mvnCmd} sonar:sonar -Dsonar.host.url=${sonarqubeUrl} -Dsonar.projectName=${JOB_BASE_NAME}-${devTag}"
    }
      
    stage("Publish Artifacts"){
      // Publish binary to Nexus for repository and later use
      sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::${nexusRelUrl}"
    }

    stage("Build & Tag Image"){
      // Delete old build definition and always exit True regardless exit code.
      sh "oc delete bc ${appName} -n ${appNamespace} || true"
      // Defining new build with new output target using Nexus mirror.
      sh "oc new-build --name=${appName} --binary=true  jboss-eap71-openshift:1.2 --to=${appName}/${appName}:${devTag} -e MAVEN_MIRROR_URL=${mavenMirrorUrl} -n ${appNamespace}"
      // Start to build the new build definition using artifact uploaded to Nexus above.
      sh "oc start-build ${appName} --follow --from-file=http://nexus3.nexus3.svc.cluster.local:8081/repository/releases/org/jboss/as/quickstarts/jboss-as-kitchensink/${version}/jboss-as-kitchensink-${version}.jar -n ${appNamespace}"
    
    stage("Blue-Green deployment stage"){
      // Determine which deployment is active
      activeSvc = sh (returnStdout: true, script: "oc get route kitchensink -n \${appNamespace} -o jsonpath='{ .spec.to.name }'").trim()
      if (activeSvc == "kitchensink-blue"){
        destSvc = "kitchensink-green"
      }
      echo "Current Activate Service:       " + activeSvc
      echo "Deployment Service:             " + destSvc 
      if (activeSvc == "kitchensink-blue"){
        // Tag green image as green latest
        sh "oc tag ${appName}:${devTag} ${appName}-green:latest -n ${appNamespace}"
        // Make sure no automatic trigger set
        sh "oc set triggers dc/${appName} --remove-all -n ${appNamespace}"
        // Set new image
        sh "oc set image dc/${appName} kitchensink=docker-registry.default.svc:5000/kitchensink/${appName}-green:latest -n ${appNamespace}"
        // Rolling new green image
        sh "oc rollout latest dc/${appName} -n ${appNamespace}"
        } else{
        // Tag blue image as blue latest
        sh "oc tag ${appName}:${devTag} ${appName}-blue:latest -n ${appNamespace}"
        // Make sure no automatic trigger set
        sh "oc set triggers dc/${appName} --remove-all -n ${appNamespace}"
        // Set new image
        sh "oc set image dc/${appName} kitchensink=docker-registry.default.svc:5000/kitchensink/${appName}-blue:latest -n ${appNamespace}"
        // Rolling new green image
        sh "oc rollout latest dc/${appName} -n ${appNamespace}"
        }
      }
    }
  }
}
// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}
