node {
    // Run the pipeline job
     
    sendBuildEvent("JOB_STARTED", null)
    try
    {  
         runPipeline()         
         sendBuildEvent("JOB_ENDED", "SUCCESS")
    }
    catch (all)
    {
         sendBuildEvent("JOB_ENDED", "FAILURE")
         error 'Pipeline job failed'
    }
}

def getRootPath() {
	def path = CI_ROOT_PATH.trim()
	if (path != "") 
	{
		path = path + '/'
	}
	return path
}

def runPipeline()
{
    
    def images = ['app': null, 'app_tests': null]
    def containers = ['app': null, 'app_tests': null]
    def root = getRootPath()

    // Pull down the git repo with the source
    checkout scm
    
    docker.withServer(env.CI_DOCKER_HOST)
    {
        try
        {
            def imagetag = "${env.BUILD_ID}"
    
            // Run the build
            stage 'run build'
            runBuild("${root}javademo", images, imagetag)
            
            // Start Docker app/test containers for integration testing
            stage 'run integration tests'
            runIntegrationTests(images, imagetag, containers)        
    
            // Publish the Docker images to a Docker registry
            stage 'push docker images'
            publishDockerImages(images, imagetag)
        }
        catch (all)
        {
            error 'Pipeline error'
        }
        finally
        {
            cleanup(images, containers)
        }    
    }
}

def runBuild(projectDirectory,  images, imagetag)
{
    sendBuildEvent("BUILD_STARTED", null)
     
    try
    {
        // Run the java build and unit tests inside a maven container
        stage 'build java artifacts'
        buildWithMavenContainer(projectDirectory, imagetag)
        
        // Build the Docker images for the app and integration test
        // from the artifacts built in the workspace
        stage 'build docker images'
        buildDockerImages(projectDirectory, images, imagetag)
    
        sendBuildEvent("BUILD_ENDED", "SUCCESS")
    }
    catch (e)
    {
        sendBuildEvent("BUILD_ENDED", "FAILURE")
        throw e
    }     
}

// Run maven build that will compile, run unit tests, and create artifact binaries
def buildWithMavenContainer(projectDirectory, imagetag)
{
    def buildContainerId = 'syndicatebuild-' + imagetag
    def buildPath = pwd() + '/' + projectDirectory
    
    echo '***** Start container to run build and unit tests'
    def maven = docker.image('maven:3.3-jdk-7')
    
    try
    {    
        // Start the Maven build container
        // NOTES:
        // This mounts a volume (with -v), inside the Maven container, 
        // containing the project source directory to build from.
        // -w sets the current work directory (-w) to the project source
        // directory.
        // The maven cache volume is stored on the Docker host and is mapped
        // into this container.
        maven.withRun('--name ' + buildContainerId +
                    ' -v ' + buildPath + ':' + buildPath +
                    ' -v /m2repo:/m2repo ' +
                    ' -w ' + buildPath +
                    ' -t --entrypoint=cat')
        {                
            // Execute the Maven build inside the container.
            // The package directive compiles the source, runs the unit test,
            // creates the artifacts.
            docker.script.sh 'docker exec -t ' + buildContainerId + ' bash -c "mvn -Dmaven.repo.local=/m2repo clean package"'
        }

        echo '***** Build and unit tests were successful'
    }
    catch (all)
    {
        error 'Build or unit test failed'
    }    
}

def buildDockerImages(projectDirectory, images, imagetag)
{
    parallel "Building Docker app image":
    {
        images.app = docker.build("${env.JOB_NAME}:${imagetag}", "${projectDirectory}/app")
    },
    "Building Docker integration test image":
    {
        images.app_tests = docker.build("${env.JOB_NAME}-tests:${imagetag}", "${projectDirectory}/integration-test")
    },
    failFast: false
    
    echo '***** Docker builds for images successful'
}

def runIntegrationTests(images, imagetag, containers)
{
    sendBuildEvent("TEST_STARTED", null)
    
    echo '***** Integration test stage...Start Docker containers for integration testing'
    try
    {
        containers.app = images.app.run("-d -i --name ${env.JOB_NAME}-${imagetag}")
        containers.app_tests = images.app_tests.run("-d -i --link ${env.JOB_NAME}-${imagetag}:app --name ${env.JOB_NAME}-tests-${imagetag}")

        echo '***** Preparing for integration testing...Wait for app and test containers to start up'
        retry(10)
        {
            sleep 3L
            docker.script.sh "docker exec -t ${env.JOB_NAME}-${imagetag} curl --write-out '\n' localhost:8080/demoapp/status"
            docker.script.sh "docker exec -t ${env.JOB_NAME}-tests-${imagetag} curl --write-out '\n' localhost:8080/demotest/status"
        }

        echo '***** Execute test on integration test container'
        // This will execute the test from inside the test container.
        // The output will be piped to a file so that it can be read into a
        // variable to process the test output.
        docker.script.sh "docker exec -t ${env.JOB_NAME}-tests-${imagetag} curl --write-out '\n' localhost:8080/demotest/test > test.txt 2>&1"
        def testoutput = docker.script.readFile('test.txt')
        echo testoutput

        if (testoutput.contains('pass: true'))
        {
            echo '***** Integration test passed.'
            sendBuildEvent("TEST_ENDED", "SUCCESS")
        }
        else
        {
            error 'Integration test stage failed'
        }
    }
    catch (all)
    {
        sendBuildEvent("TEST_ENDED", "FAILURE")
        
        // Force build failure.
        error 'Integration test stage failed'
    }
}

def cleanup(images, containers) {

     parallel "Stop demo app container":
     {
          try {
               containers.app.stop()
               docker.script.sh "docker rmi ${images.app.id}"
          }
          catch (all) {echo 'Error stopping demo app container'}
     },
     "Stop integration tests container":
     {
          try {
               containers.app_tests.stop()
               docker.script.sh "docker rmi ${images.app_tests.id}"
          }
          catch (all) {echo 'Error stopping integration tests container'}
     },
     failFast: false
}

def publishDockerImages(images, imagetag) {

     sendBuildEvent("PUBLISH_STARTED", null)

     try {
          docker.withRegistry(env.CI_IMAGE_REGISTRY_URL, 'docker-registry-login') {
               
               images.app.push(imagetag)
               images.app_tests.push(imagetag)
          }
          
          sendBuildEvent("PUBLISH_ENDED", "SUCCESS")
     }
     catch (all) {
          sendBuildEvent("PUBLISH_ENDED", "FAILURE")
          echo 'Failed to tag/push to VDR image'
          error 'Failed to tag/push to VDR image'
     }
}

/**
 * Sends a build event to the build service.
 * @param type Stage event type
 * @param result Result for the stage.  Required for ENDED event types
 */
def sendBuildEvent(type, result)
{
    // Use the Platform Service Registry to communicate with the Build Service.
 
    // Get the event callback URL from the Jenkins job parameter.
    // Test existence of the parameter by trying to access it.
    def eventCallbackUrl
    try
    {
        eventCallbackUrl = CI_EVENT_CALLBACK.trim()
        if (eventCallbackUrl == "")
        {
            throw new Exception()
        }
    }
    catch (all)
    {
        echo 'Not sending build event since callback URL is not set'
        return    
    }
    
    // Create the URL for the Build Service Event REST API
    def serviceRegistry = env.SERVICE_REGISTRY_HOSTNAME
    def buildServiceServiceRegistryPath =
        (env.BUILD_SERVICE_SR_PATH || env.BUILD_SERVICE_SR_PATH == "" ?
            env.BUILD_SERVICE_SR_PATH : "/default/ci/buildservice")
    
    def buildServiceUrl = "http://" + serviceRegistry + buildServiceServiceRegistryPath
    def buildServiceApiPath = "/jobs/${env.JOB_NAME}/builds/${env.BUILD_ID}/events" 
    def buildServiceEventUrl = buildServiceUrl + buildServiceApiPath
    
    def jsonEventPayload =
        "{ " +
            "\"event\": {" + 
                "\"type\": \"${type}\", " +
                "\"callback\": \"${eventCallbackUrl}\"" +
                (result ? ", \"result\": \"${result}\"" : "") +
            " } " +
        "}"
       
    // Send event payload to build service.
    // Don't fail the job if the event can't be sent 
    try
    {
        sh "curl -sS -X POST -H \"Content-Type: application/json\" -d '" +
            jsonEventPayload + "' " +  buildServiceEventUrl
        echo 'Sent build event "' + type + '" to Build Service URL: ' + buildServiceEventUrl         
    }
    catch (all)
    {
        echo 'Failed to send build event "' + type + '" to Build Service URL: ' + buildServiceEventUrl
    }
}
