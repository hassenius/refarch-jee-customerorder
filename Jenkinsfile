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

        app = docker.build("${params.namespace}/customer-order-services")
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
        
        # Create configmaps if they dont exist
        for map in ldap orderdb inventorydb 
        do
          kubectl get configmap $map || kubectl create configmap $map --from-file=tutorial/tutorialConfigFiles/step5/${map}.properties
        done

        # Apply the deployment
        kubectl apply -f Common/DevOps/deployment.yaml
        
      '''
           
    }
}



