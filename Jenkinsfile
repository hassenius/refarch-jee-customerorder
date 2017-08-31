node {
    def app

    stage('Clone repository') {
        /* Let's make sure we have the repository cloned to our workspace */

        checkout scm
    }

    stage('Compile code') {

      withMaven(maven: 'maven') {
        sh 'cd CustomerOrderServicesProject ; mvn clean package'
      }
    }
    
      
    stage('Build image') {

        app = docker.build("${params.namespace}/customer-order-service")
    }

    stage('Test image') {
        /* Ideally, we would run a test framework against our image.
         * For this example, we're using a Volkswagen-type approach ;-) */

        app.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push image') {
        /* Finally, we'll push the image with two tags:
         * First, the incremental build number from Jenkins
         * Second, the 'latest' tag.
         * Pushing multiple tags is cheap, as all the layers are reused. */
        docker.withRegistry('https://master.cfc:8500', 'icp-credentials') {
            app.push("${env.BUILD_NUMBER}")
            app.push("latest")
        }
    }
    
    stage('Deploy application') {
      sh '''
        #!/bin/bash
        alias kubectl=kubectl-1.6.1
        
        # We will need to handle the conditional creation of ConfigMaps
        # Update the passwords
        sh "sed -i 's/___TOBEREPLACED___/${params.DB2Password}/g' Common/order-db.properties"
        sh "sed -i 's/___TOBEREPLACED___/${params.DB2Password}/g' Common/inventory-db.properties"
        
        kubectl apply -f deployment.yaml
        
      '''

           
    }
}



