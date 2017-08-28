# Building and deploying 

## Prerequisites
* kubectl
* Docker CLI

Note: 
Currently this works from cfc-master.
need to sort out some ca stuff before docker and kubctl will work properly from laptop or skytap desktop VM


## Preparation steps

### Create user and namespace
#### Creating a namespace
1. Login as a system administrator.
1. From the navigation menu, select System > Namespaces.
1. Click New Namespace.
1. Enter a namespace name "websphere'.
1. Click Add Namespace.
 
#### Adding a user to a namespace

1. Log on as a system administrator.
1. From the navigation menu, select System > Users.
1. Click New User.
1. Enter "user1" as the user name, and provide a password.
1. Select Namespace "websphere".

# Containerize application

After the application has been compiled, we can build a new container image using the IBM WebSphere-Liberty image from Docker Hub as a base, adding our configurations and application on top.
This image will be stored in the IBM Cloud Private image repository, 

1. Login to the ICp image repository using the credentials created above

   ``` $ docker login master.cfc:8500 ```


1. Create a ```Dockerfile``` with the following content

    ```
    FROM websphere-liberty:webProfile7

    COPY Common/server.xml /config
    COPY Common/server.env.remote /config/server.env
    COPY CustomerOrderServicesApp/target/CustomerOrderServicesApp-0.1.0-SNAPSHOT.ear /config/apps

    RUN /opt/ibm/wlp/bin/installUtility install  --acceptLicense \
        appSecurity-2.0 \
        ejbLite-3.1 \
        ldapRegistry-3.0 \
        localConnector-1.0 \
        jaxrs-1.1 \
        jdbc-4.1 \
        jpa-2.0 \
        jsp-2.3 \
        servlet-3.1

    ADD https://artifacts.alfresco.com/nexus/content/repositories/public/com/ibm/db2/jcc/db2jcc4/10.1/db2jcc4-10.1.jar /db2lib/db2jcc4.jar
    ADD http://download.osgeo.org/webdav/geotools/com/ibm/db2jcc_license_cu/9/db2jcc_license_cu-9.jar /db2lib/db2jcc_lisence_cu.jar
    ```

1. Build the docker image

    ``` $ docker build -t master.cfc:8500/websphere/customer-order-services:0.1 . ```

1. Push the image to the ICp Image Repository

    ``` $ docker push master.cfc:8500/websphere/customer-order-services:0.1 ```

 
# Deploy application

### From GUI
From navigation menu, select Applications.
Select Deploy Application.
On the General tab, provide an application name.
On the Container Settings tab, provide a container name, image name, and port.​​​​​​
Application name: customerorderservices
Container name: customerorderservices
Image name: master.cfc:8500/websphere/customer-order-services:0.1
Container port: 9080
Select Deploy.

right click the cog wheel for the application and select Expose
Expose Method: ClusterIP
Port: 80
Target Port: 9080

### From kubectl
1. Generate deployment.yaml

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: customerorderservices
      labels:
        app: customerorderservices
    spec:
      ports:
        - port: 80
          targetPort: 9080
      selector:
        app: customerorderservices
      type: ClusterIP
      
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      labels:
        app: customerorderservices
      name: customerorderservices
    spec:
      replicas: 1
      strategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 1
        type: RollingUpdate
      template:
        metadata:
          labels:
            app: customerorderservices
        spec:
          containers:
          - image: master.cfc:8500/websphere/customerorderservices:0.1
            imagePullPolicy: Always
            name: customerorderservices
            ports:
            - containerPort: 9080
              protocol: TCP
            resources: {}
            terminationMessagePath: /dev/termination-log
            terminationMessagePolicy: File
          dnsPolicy: ClusterFirst
          imagePullSecrets:
          - name: admin.registrykey
          restartPolicy: Always
          schedulerName: default-scheduler
          securityContext: {}
          terminationGracePeriodSeconds: 30
    ```

1. Run kubectl to apply the deployment

    ```kubectl apply -f deployment.yaml```
