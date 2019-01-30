/* branch name is env.BRANCH_NAME */

DOCKER_IMAGE_TAG = "${env.BUILD_TIMESTAMP}"

node {
    def docker_image

    env.DOCKER_IMAGE_NAMESPACE_DEV      = "${env.BRANCH_NAME}-dev"
    env.DOCKER_IMAGE_NAMESPACE_PROD     = "${env.BRANCH_NAME}-prod"
    env.DOCKER_IMAGE_APP_REPOSITORY     = "atsea-appserver"
    env.DOCKER_IMAGE_DB_REPOSITORY      = "atsea-database"
    env.DOCKER_REGISTRY_HOSTNAME        = "dtr.west.us.se.dckr.org"
    env.DOCKER_REGISTRY_URI             = "https://${env.DOCKER_REGISTRY_HOSTNAME}"
    env.DOCKER_UCP_HOSTNAME        = "ucp.west.us.se.dckr.org"
    env.DOCKER_UCP_URI             = "https://${env.DOCKER_UCP_HOSTNAME}"
    env.DOCKER_REGISTRY_CREDENTIALS_ID  = "jenkins"

    stage('Clone') {
        /* Let's make sure we have the repository cloned to our workspace */
        checkout scm
    }

    stage('Build') {
        /* This builds the actual image; synonymous to
        * docker build on the command line */
        dir ("app") {
            docker_app_image = docker.build("${env.DOCKER_IMAGE_NAMESPACE_DEV}/${env.DOCKER_IMAGE_APP_REPOSITORY}")
        }
        dir ("database") {
            docker_db_image = docker.build("${env.DOCKER_IMAGE_NAMESPACE_DEV}/${env.DOCKER_IMAGE_DB_REPOSITORY}")
        }
    }

    stage('Test') {
        /* Ideally, we would run a test framework against our image.
        * For this example, we're using a Volkswagen-type approach ;-) */

        docker_app_image.inside {
            sh 'echo "App Tests passed"'
        }
        docker_db_image.inside {
            sh 'echo "DB Tests passed"'
        }
    }

    stage('Push') {
        /* Finally, we'll push the image with two tags:
        * First, the incremental build number from Jenkins
        * Second, the 'latest' tag.
        * Pushing multiple tags is cheap, as all the layers are reused. */

        /* Get Token */
        withCredentials([usernamePassword(credentialsId: env.DOCKER_REGISTRY_CREDENTIALS_ID, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            def token_response = httpRequest acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${env.DOCKER_UCP_URI}/auth/login", requestBody: "{  \"password\": \"${PASSWORD}\",  \"username\": \"${USERNAME}\"}"
            def token_result = readJSON text: token_response.content
            def token = token_result.auth_token
        }

        /* Create Organizations */
        println('Token response: ' + token_response)
        println('Token result: ' + token_result)
        println('Token: ' + token)
        httpRequest acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${env.DOCKER_UCP_URI}/accounts", requestBody: "{  \"fullName\": \"${env.DOCKER_IMAGE_NAMESPACE_DEV}\",  \"isOrg\": true,  \"name\": \"${env.DOCKER_IMAGE_NAMESPACE_DEV}\" }", customHeaders: [[name: 'Authorization', value: "Bearer ${token}"]]
        httpRequest acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${env.DOCKER_UCP_URI}/accounts", requestBody: "{  \"fullName\": \"${env.DOCKER_IMAGE_NAMESPACE_PROD}\",  \"isOrg\": true,  \"name\": \"${env.DOCKER_IMAGE_NAMESPACE_PROD}\" }", customHeaders: [[name: 'Authorization', value: "Bearer ${token}"]]

        /* Create Repositories */
        httpRequest acceptType: 'APPLICATION_JSON', authentication: env.DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${env.DOCKER_REPOSITORY_URI}/api/v0/repositories/${env.DOCKER_IMAGE_NAMESPACE_DEV}", requestBody: "{ \"enableManifestLists\": true, \"immutableTags\": false, \"longDescription\": \"App Server for AtSea app\", \"name\": \"${env.DOCKER_IMAGE_APP_REPOSITORY}\", \"scanOnPush\": false, \"shortDescription\": \"App Server for AtSea app\", \"tagLimit\": 0, \"visibility\": \"public\"}"
        httpRequest acceptType: 'APPLICATION_JSON', authentication: env.DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${env.DOCKER_REPOSITORY_URI}/api/v0/repositories/${env.DOCKER_IMAGE_NAMESPACE_DEV}", requestBody: "{ \"enableManifestLists\": true, \"immutableTags\": false, \"longDescription\": \"Database Server for AtSea app\", \"name\": \"${env.DOCKER_IMAGE_DB_REPOSITORY}\", \"scanOnPush\": false, \"shortDescription\": \"Database Server for AtSea app\", \"tagLimit\": 0, \"visibility\": \"public\"}"

        /* Push Docker images */
        docker.withRegistry(env.DOCKER_REGISTRY_URI, env.DOCKER_REGISTRY_CREDENTIALS_ID) {
            docker_app_image.push(DOCKER_IMAGE_TAG)
            docker_db_image.push(DOCKER_IMAGE_TAG)
        }
    }

    stage('Scan') {
        /* App Image Scanning */
        httpRequest acceptType: 'APPLICATION_JSON', authentication: env.DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${env.DOCKER_REGISTRY_URI}/api/v0/imagescan/scan/${env.DOCKER_IMAGE_NAMESPACE_DEV}/${env.DOCKER_IMAGE_APP_REPOSITORY}/${DOCKER_IMAGE_TAG}/linux/amd64"

        def scan_result

        def scanning = true
        while(scanning) {
            def scan_result_response = httpRequest acceptType: 'APPLICATION_JSON', authentication: env.DOCKER_REGISTRY_CREDENTIALS_ID, httpMode: 'GET', ignoreSslErrors: true, responseHandle: 'LEAVE_OPEN', url: "${env.DOCKER_REGISTRY_URI}/api/v0/imagescan/repositories/${env.DOCKER_IMAGE_NAMESPACE_DEV}/${env.DOCKER_IMAGE_APP_REPOSITORY}/${DOCKER_IMAGE_TAG}"
            scan_result = readJSON text: scan_result_response.content

            if (scan_result.size() != 1) {
                println('Response: ' + scan_result)
                error('More than one imagescan returned, please narrow your search parameters')
            }

            scan_result = scan_result[0]

            if (!scan_result.check_completed_at.equals("0001-01-01T00:00:00Z")) {
                scanning = false
            } else {
                sleep 1
            }

        }
        println('App Response JSON: ' + scan_result)

        /* DB Image Scanning */
        httpRequest acceptType: 'APPLICATION_JSON', authentication: env.DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${env.DOCKER_REGISTRY_URI}/api/v0/imagescan/scan/${env.DOCKER_IMAGE_NAMESPACE_DEV}/${env.DOCKER_IMAGE_DB_REPOSITORY}/${DOCKER_IMAGE_TAG}/linux/amd64"

        scanning = true
        while(scanning) {
            def scan_result_response = httpRequest acceptType: 'APPLICATION_JSON', authentication: env.DOCKER_REGISTRY_CREDENTIALS_ID, httpMode: 'GET', ignoreSslErrors: true, responseHandle: 'LEAVE_OPEN', url: "${env.DOCKER_REGISTRY_URI}/api/v0/imagescan/repositories/${env.DOCKER_IMAGE_NAMESPACE_DEV}/${env.DOCKER_IMAGE_DB_REPOSITORY}/${DOCKER_IMAGE_TAG}"
            scan_result = readJSON text: scan_result_response.content

            if (scan_result.size() != 1) {
                println('Response: ' + scan_result)
                error('More than one imagescan returned, please narrow your search parameters')
            }

            scan_result = scan_result[0]

            if (!scan_result.check_completed_at.equals("0001-01-01T00:00:00Z")) {
                scanning = false
            } else {
                sleep 1
            }

        }
        println('DB Response JSON: ' + scan_result)
    }

    stage('Promote') {
        httpRequest acceptType: 'APPLICATION_JSON', authentication: env.DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, requestBody: "{\"targetRepository\": \"${env.DOCKER_IMAGE_NAMESPACE_PROD}/${env.DOCKER_IMAGE_APP_REPOSITORY}\", \"targetTag\": \"${DOCKER_IMAGE_TAG}\"}", responseHandle: 'NONE', url: "${env.DOCKER_REGISTRY_URI}/api/v0/repositories/${env.DOCKER_IMAGE_NAMESPACE_DEV}/${env.DOCKER_IMAGE_APP_REPOSITORY}/tags/${DOCKER_IMAGE_TAG}/promotion"
        httpRequest acceptType: 'APPLICATION_JSON', authentication: env.DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, requestBody: "{\"targetRepository\": \"${env.DOCKER_IMAGE_NAMESPACE_PROD}/${env.DOCKER_IMAGE_DB_REPOSITORY}\", \"targetTag\": \"${DOCKER_IMAGE_TAG}\"}", responseHandle: 'NONE', url: "${env.DOCKER_REGISTRY_URI}/api/v0/repositories/${env.DOCKER_IMAGE_NAMESPACE_DEV}/${env.DOCKER_IMAGE_DB_REPOSITORY}/tags/${DOCKER_IMAGE_TAG}/promotion"
    }

    // stage('Deploy') {
    //     withDockerServer([credentialsId: env.DOCKER_UCP_CREDENTIALS_ID, uri: env.DOCKER_UCP_URI]) {
    //         sh "docker service update --image ${env.DOCKER_REGISTRY_HOSTNAME}/${env.DOCKER_IMAGE_NAMESPACE_PROD}/${env.DOCKER_IMAGE_REPOSITORY}:${DOCKER_IMAGE_TAG} ${env.DOCKER_SERVICE_NAME}" 
    //     }
    // }

}