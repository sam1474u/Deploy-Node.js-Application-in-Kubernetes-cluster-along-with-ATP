A simple backend node.js application on Oracle Container Engine for Kubernetes(OKE) which connects to OCI Autonomous transaction processing(ATP) instance created via OCI service broker
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

In this tutorial we will create a simple backend node.js application on Oracle Container Engine for Kubernetes(OKE) which connects to OCI Autonomous transaction processing(ATP) instance created via OCI service broker.
We will use oracledb node.js package and oracle instant client for connecting to ATP. We will also create a docker container for DB app built using node.js and store the container image on Oracle Container Registry(OCIR). And finally, expose the deployment (by creating a load balancer service) and test the functionality.


1. High level overview of the steps

        - Create OKE cluster
        - Deploy service catalog and OCI service broker
        - Provision ATP service instance using service broker
        - Using a temp pod, Manually bootstrap ATP instance (create a schema and table)
        - Build the application docker image and push to OCIR . This container will run the node.js service.
        - Create a kubernetes deployment using this image.
        - Expose the pod (Create service)
        - Test

 
2. Architecture

     ![image](https://user-images.githubusercontent.com/42166489/109262025-d99d7f00-7826-11eb-9acd-3fa2e5b787f9.png)


3. Create OKE cluster

   please refer https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-full/index.html
   
4. Deploy service catalog and OCI service broker

    1. Install Service Catalog

           helm repo add svc-cat https://kubernetes-sigs.github.io/service-catalog
        
           helm install catalog svc-cat/catalog

    2. Deploy OCI Service Broker

            https://github.com/oracle/oci-service-broker/releases/download/v1.5.2/oci-service-broker-1.5.2.tgz
            
            
    3. OCI credentials

        To generate secret, you can run this as a .sh file. Collect all the deails, if passphrase is not given during key generation, keep as blank. 
        The private key is your .pem file which you generate. 

                  kubectl create secret generic ocicredentials \
                   --from-literal=tenancy=<CUSTOMER_TENANCY_OCID> \
                   --from-literal=user=<USER_OCID> \
                   --from-literal=fingerprint=<USER_PUBLIC_API_KEY_FINGERPRINT> \
                   --from-literal=region=<USER_OCI_REGION> \
                   --from-literal=passphrase=<PASSPHRASE_STRING> \
                   --from-file=privatekey=<PATH_OF_USER_PRIVATE_API_KEY>
                   
                   
    4. Quick Setup of Service Broker


             helm install oci-service-broker https://github.com/oracle/oci-service-broker/releases/download/v1.5.2/oci-service-broker-1.5.2.tgz \
              --set ociCredentials.secretName=ocicredentials \
              --set storage.etcd.useEmbedded=true \
              --set tls.enabled=false
              
          // insert image here.
            
    5. RBAC Permissions for registering OCI Service Broker

           kubectl create clusterrolebinding cluster-admin-brokers --clusterrole=cluster-admin --user=<USER_ID>
           
    6.  Register OCI Service Broker
            
            curl -LO https://github.com/oracle/oci-service-broker/releases/download/v1.5.2/oci-service-broker-1.5.2.tgz | tar xz
            
            # Ensure <NAMESPACE_OF_OCI_SERVICE_BROKER> is replaced with the a proper namespace in oci-service-broker.yaml
              kubectl create -f oci-service-broker/samples/oci-service-broker.yaml
            
         Get the status of the broker:
         
                ./svcat get brokers
         
         ![image](https://user-images.githubusercontent.com/42166489/109262109-f6d24d80-7826-11eb-9f27-319fcd17701f.png)
         
         Get Services List
         
                svcat get classes
                
          ![image](https://user-images.githubusercontent.com/42166489/109262149-0356a600-7827-11eb-85e8-e7b7221c26b9.png)
          
         Get Service Plans
         
          ![image](https://user-images.githubusercontent.com/42166489/109262153-05b90000-7827-11eb-9c6d-99895189b0f7.png)
         
         verify the service broker status is ready
         
              kubectl get clusterservicebrokers
         
5.   Provision ATP service instance using service broker

      High level steps:
      
              Create a secret
              Create ATP serviceinstance
              Create ATP servicebinding
              
        ![image](https://user-images.githubusercontent.com/42166489/109262203-21240b00-7827-11eb-82f6-f95f2d06d3b0.png)
       
      Create a file named atp-secret.yaml with the following details: 
      
          Please note : the ATP DB password and wallet password are encoded. you can use any online encoder or can use command. The both password can be same also for ease.
      

                            apiVersion: v1
                            kind: Secret
                            metadata:
                              name: atp-secret
                            data:
                              # {"password":"s123456789S@"}
                              password: eyJwYXNzd29yZCI6InMxMjM0NTY3ODlTQCJ9
                              # {"walletPassword":"Welcome_123"}
                              walletPassword: eyJ3YWxsZXRQYXNzd29yZCI6IldlbGNvbWVfMTIzIn0K

         
       Create a file named atp-instance.yaml with the following details: 
       

                              apiVersion: servicecatalog.k8s.io/v1beta1
                              kind: ServiceInstance
                              metadata:
                                name: demodb
                              spec:
                                clusterServiceClassExternalName: atp-service
                                clusterServicePlanExternalName: standard
                                parameters:
                                  name: demodb
                                  compartmentId: "ocid1.compartment.oc1..xxxx"
                                  dbName: demodb
                                  cpuCount: 1
                                  storageSizeTBs: 1
                                  licenseType: NEW
                                  autoScaling: false
                                  freeFormTags:
                                    testtag: demodb
                                parametersFrom:
                                  - secretKeyRef:
                                      name: atp-secret
                                      key: password
                                      
                                      
      Create a file named atp-binding.yaml with the following details: 
      
                              apiVersion: servicecatalog.k8s.io/v1beta1
                              kind: ServiceBinding
                              metadata:
                                name: atp-demo-binding
                              spec:
                                instanceRef:
                                  name: demodb
                                parametersFrom:
                                  - secretKeyRef:
                                      name: atp-secret
                                      key: walletPassword


      Apply the config
       
             kubectl apply -f atp-secret.yaml 
             kubectl apply -f atp-instance.yaml 
             kubectl apply -f atp-binding.yaml              
             
             
      Ensure that service instance is in Ready state: 
      
                  kubectl get serviceinstance
                  
       ![image](https://user-images.githubusercontent.com/42166489/109262219-2a14dc80-7827-11eb-8ade-5f86ae07af8c.png)
      
      atp-demo-binding secret automatically gets created which has the wallet details (tnsnames.ora, cwallet.sso etc.) of ATP database we created. 
      we are now ready to use it in our application.
      
 6. Building a Node.js app

    Once the DB is initialized and ready, time to create our db-app docker image. lets automate this process to get a different perspective.
    This is a very basic node.js file, which uses oracledb package to connect to database. There are more advanced examples, refer the link in references section.

    Create a project folder named db-app and create the following files:
    Create a file named dbconfig.js and add the following details: 
    
            module.exports = {
            user : process.env.OADB_USER ,
            password : process.env.OADB_PW,
            connectString : process.env.OADB_SERVICE,
            externalAuth : process.env.NODE_OADB_EXTERNALAUTH ? true : false
            };
            
            
     Create a file named db.js and add the following details: 
     
                                                
                                const http = require('http')
                                const oracledb = require('oracledb');
                                const dbconfig = require('./dbconfig.js');

                                let connection;

                                const server = http.createServer(async (req, res) => {
                                   //console.log(req.url)
                                   res.setHeader('Content-Type','text/html');

                                   if (req.url === '/home'){
                                    res.write("In home page, Welcome...")
                                    res.end()
                                   } 

                                   if (req.url === '/products') {
                                    console.log("In products page...")

                                    try {

                                      connection =  await oracledb.getConnection(dbconfig);
                                      const result =  await connection.execute(
                                      `SELECT brand, title, description
                                       FROM products`
                                    );

                                    //console.log(result.rows);
                                    res.write('The results fetched from DB are :' + '

                                ');
                                    for(let results in result.rows){
                                        const [brand,title,description] = result.rows[results];
                                        console.log(brand,title,description);
                                        res.write('Brand: '+ brand + ' Title: '+ title +' Description: '+ description);
                                        res.write('
                                ')
                                    }

                                    res.end()

                                  } catch (err) {
                                    console.error(err);
                                  } finally {
                                    if (connection) {
                                    try {
                                      await connection.close();
                                    } catch (err) {
                                      console.error(err);
                                      }
                                     }
                                    }
                                }
                                });

                                server.listen(process.env.PORT);

                                function terminate(){
                                  console.log("\nTerminating");
                                    if (connection) {
                                      try {
                                        connection.close();
                                      } catch (err) {
                                        console.error(err);
                                        }
                                    }
                                    process.exit(0);
                                }
                                process
                                  .on('SIGTERM',terminate)
                                  .on('SIGINT', terminate)
      
      
      Create a file named package.json and add the following details: 
      
                                {
                          "name": "db-app",
                          "version": "1.1.0",
                          "description": "A Simple DB application",
                          "main": "db.js",
                          "dependencies": {
                              "oracledb": "^4.2.0"
                          }
                      }
      
     Create a Dockerfile, In which we are trying to create a container with oracle-instantclient19.5, node.js and all its dependencies. Application will be listening on 8080.
     Create a file named Dockerfile and add the following details: 
     
                 FROM oraclelinux:7-slim as db-app

                 RUN yum -y upgrade && \
                  yum -y update && \
                  yum -y install oracle-release-el7 && \
                  yum-config-manager --enable ol7_oracle_instantclient && \
                  yum -y install oracle-instantclient19.5-sqlplus && \
                  yum -y install oracle-instantclient19.5-tools && \
                  yum -y install oracle-nodejs-release-el7 oracle-release-el7 && \
                  yum install -y nodejs 

                RUN mkdir -p /home/node/app
                WORKDIR /home/node/app
                COPY * ./
                RUN npm install

                ENV NODE_ENV "production"
                ENV PORT 8080
                ENV ORACLE_HOME /usr/lib/oracle/19.5/
                RUN export PATH=$ORACLE_HOME:$PATH
                RUN export PATH=/usr/lib/oracle/19.5/client64/bin/:$PATH
                ENV LD_LIBRARY_PATH /usr/lib/oracle/19.5/
                EXPOSE 8080

                CMD [ "node", "db.js" ]
                
                
       Build the image as below:
       
       docker build -t syd.ocir.io/your_tenancy_namespace/db-app:latest .
       
       To push an image to OCIR, you need to login to the registry(using your user id and auth token)
       
               $ docker login syd.ocir.io
               $ docker push syd.ocir.io/your_tenancy_namespace/db-app:latest
       
       
       ![image](https://user-images.githubusercontent.com/42166489/109262312-54ff3080-7827-11eb-8175-b106b13cb791.png)

       ![image](https://user-images.githubusercontent.com/42166489/109262325-5af51180-7827-11eb-94bc-159f8e0154b9.png)
       
       Moving on, The DB POD we create next would also need access to schema user and password created in the previous step. 
       We also specify the service name to connect to (refering to the tnsnames.ora) 
       
           kubectl create secret generic atp-demo-credentials --from-literal=oadb_service=demodb_tp --from-literal=oadb_user='test_user' --from-literal=oadb_pw='default_Password1'
       
       
       To pull an image from OCIR registry, we also need to create docker-registry secret which will be used in our deployment spec: 
       
       
            kubectl create secret docker-registry secret-name --docker-server=region-key.ocir.io --docker-username='tenancy-namespace/oci-username'
            --docker-password='oci-auth-token'         
            --docker-email='email-address
       
       
 7. Finally, create Application Deployment

     Create a file named db-app.yaml and add the following details: 
     
                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: db-app
                spec:
                  replicas: 1
                  selector:
                    matchLabels:
                      name: db-app
                  template:
                    metadata:
                      labels:
                        name: db-app
                    spec:
                        initContainers:
                          - name: decode-wallet
                            image: oraclelinux:7-slim
                            command: ["/bin/sh","-c"]
                            args: 
                            - for i in `ls -1 /tmp/wallet | grep -v user_name`; do cat /tmp/wallet/$i  | base64 --decode > /wallet/$i; done; ls -l /wallet/*;sleep 10s;
                            volumeMounts:
                            - name: wallet-raw
                              mountPath: /tmp/wallet
                            - name: wallet
                              mountPath: /wallet
                        containers:
                          - name: db-app
                            image: syd.ocir.io/ociateam/db-app:latest
                            imagePullPolicy: Always
                            env:
                            - name: OADB_USER
                              valueFrom:
                                secretKeyRef:
                                  name: atp-demo-credentials
                                  key: oadb_user
                            - name: OADB_PW
                              valueFrom:
                                secretKeyRef:
                                  name: atp-demo-credentials
                                  key: oadb_pw
                            - name: OADB_SERVICE
                              valueFrom:
                                secretKeyRef:
                                  name: atp-demo-credentials
                                  key: oadb_service        
                            volumeMounts:
                              - name: wallet
                                mountPath: /usr/lib/oracle/19.5/client64/lib/network/admin/
                        volumes:
                          - name: wallet-raw
                            secret:
                              secretName: atp-demo-binding
                          - name: wallet
                            emptyDir: {}
                        imagePullSecrets:
                          - name: ocirsecret



      Apply the configuration: 
      
             kubectl apply -f db-app.yaml


8. kubectl apply -f db-app.yaml

            kubectl expose deployment db-app --port=80 --target-port=8080 --type=LoadBalancer 
            
    ![image](https://user-images.githubusercontent.com/42166489/109262406-7eb85780-7827-11eb-99dc-edc38a625186.png)
            
9. Testing

    ![image](https://user-images.githubusercontent.com/42166489/109263938-2d5d9780-782a-11eb-94dd-0c3d281ba2b6.png)       
       
