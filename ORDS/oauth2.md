# OAUTH authentication of ORDS.

## Introduction

In this lab,
- You will create ORDS roles and privileges.
- Create ORDS Client
- Create mapping between roles & privileges with the Client.
- Run the application with the OAUTH token.

Estimated time: ~45 minutes.

### Understanding Oauth

There are several methods of authorizing to the web service using OAuth. The one described here is a two-legged Oauth. OAuth revolves around registering clients, which represent a person or an application wanting to access the resource, then associating those clients to roles. Once the client is authorized, it has access to the protected resources associated with the roles.


### Prerequisites

- This lab requires the completion of lab 2 and have a working ORDS url.


## Task 1: ## Task 1: Connect to sqldevweb and Create a role.
To protect the web service, we need to create a role with an associated privilege, then map the privilege to the web service. Normally, we would expect a role to be a collection of privileges, and of course a single privilege can be part of multiple roles, but in this case we will keep it simple. The following code creates a new role called "emp_role".

1. Create a ORDS role
	```
	<copy>
  BEGIN
    ORDS.create_role(p_role_name => 'ords_role');

    COMMIT;
  END;
  /
  </copy>
	```

2. verify

 ```
 <copy>
 SELECT id, name FROM   user_ords_roles ;
 </copy>
 ```

## Task 2: Create a ORDS Privilege

1. Create privilege for ords_role and add the application url pattern 'rest-fullstack'

     ```
     <copy>
      DECLARE
       l_roles_arr    OWA.vc_arr;
       l_patterns_arr OWA.vc_arr;
      BEGIN
       l_roles_arr(1)    := 'ords_role';
       l_patterns_arr(1) := '/rest-fullstack/*';

       ORDS.define_privilege (
         p_privilege_name => 'ords_priv',
         p_roles          => l_roles_arr,
         p_patterns       => l_patterns_arr,
         p_label          => 'ords Data',
         p_description    => 'Allow access to the ORDS data.'
       );

       COMMIT;
      END;
      /
      </copy>
  	 ```

2. verify the mappings for role to privilege. Note a single privilege can map an array  of patterns or the ORDS URL.
     ```
     <copy>
     SELECT privilege_id, privilege_name, role_id, role_name
      FROM   user_ords_privilege_roles where role_name='ords_role';

      SELECT r.privilege_id, privilege_name, role_id, role_name, pattern
      FROM   user_ords_privilege_roles r, user_ords_privilege_mappings m
      where  r.privilege_id=m.privilege_id;
     </copy>
  	 ```


3.  verify that you cannot access your url which you created in the previous lab.
    Example.
    ```
    curl --location https://g1ec399e4b6fed3-mtdrdb.adb.ap-sydney-1.oraclecloudapps.com/ords/admin/rest-fullstack/rc1/47/2000000/7

      {
          "code": "Unauthorized",
          "message": "Unauthorized",
          "type": "tag:oracle.com,2020:error/Unauthorized",
          "instance": "tag:oracle.com,2020:ecid/2edfed736c75c25a0ac520cc62c8e111"
      }
    ```
     If you access the url through a browser. you will get "401 Unauthorized" message.

   ![](images/unauth.png " ")


## Task 3: Create OAuth  Client Credentials.
   The client credentials flow is a two-legged process that seems the most natural to me as I mostly deal with server-server communication, which should have no human interaction. For this flow we use the client credentials to return an access token, which is used to authorize calls to protected resources. The example steps through the individual calls, but in reality it would be automated by the application.



  1. Run image locally and verify the image is running.

     ```
     <copy>
     BEGIN
      OAUTH.create_client(
        p_name            => 'ords_client',
        p_grant_type      => 'client_credentials',
        p_owner           => 'My Company Limited',
        p_description     => 'A client for ORDS management',
        p_support_email   => 'tim@example.com',
        p_privilege_names => 'ords_priv'
      );

      COMMIT;
    END;
    /
     </copy>
     ```

  2. Save the client_id and client_secret created. This will be required to update any application that needs to access the ORDS.

     ```
     <copy>
     SELECT id, name, client_id, client_secret  FROM   user_ords_clients;	 
     </copy>


     ID           NAME        CLIENT_ID                   CLIENT_SECRET
     ________     ___________ ___________________________ ___________________________
     10099        ords_client AKRRU8Ov4XwXC0oXrY-A9Q..    bWyap8QBX3VTlhLSyKI1FA..    

     ```

## Task 4: map the client to the role.


     ```
     <copy>
     BEGIN
       OAUTH.grant_client_role(
         p_client_name => 'ords_client',
         p_role_name   => 'ords_role'
       );

       COMMIT;
     END;
      /

     </copy>
     ```

  2.  verify
  ```
  <copy>
    SELECT client_name, role_name
    FROM   user_ords_client_roles;

  </copy>
  ```

   ![](images/registry_root_compartment.png " ")

3. Mark Access as Public  (if Private)  
   (**Actions** > **Change to Public**):

   ![](images/Public-access.png " ")



## Task 5: Deploy on Kubernetes and Check the Status

1. Verify the todo.yaml file.
   Ensure you have the image name in oracle docker registory, the name of the imagePullSecret that was created in step 5 of lab1.

	```
	<copy>cd ~/mtdrworkshop/python/Todo-List-Dockerized-Flask-WebApp;
	      cp todo_template.yaml todo.yaml
        sed -i "s|%DOCKER_REGISTRY%|${DOCKER_REGISTRY}|g" todo.yaml
        kubectl create -f todo.yaml
  </copy>
	```

2. Check the status using the following commands. Verify the status is running for pods and you have a external-ip for LoadBalancer. You may have to rerun the command as it could take a couple of minutes to allocate a ip-address.
    ```
     $ <copy>
      kubectl get all </copy>
     $ kubectl get all
    NAME                                      READY   STATUS             RESTARTS   AGE
    pod/todo-deployment-657895dd59-qd89j      1/1     Running            0          3m1s

    NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)          AGE
    service/kubernetes     ClusterIP      10.96.0.1     <none>           443/TCP          32h
    service/todo-service   LoadBalancer   10.96.77.65   132.226.36.134   8080:31093/TCP   3m1s

    NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/todo-deployment      1/1     1            1           3m2s

    NAME                                            DESIRED   CURRENT   READY   AGE
    replicaset.apps/todo-deployment-657895dd59      1         1         1       3m2s

    ```


	The following command returns the Kubernetes service of ToDo application with a load balancer exposed through an external API
	```
	<copy>kubectl get services</copy>
	```

	![](images/K8-service-Ext-IP.png " ")

3. $ kubectl get pods
	```
	<copy>kubectl get pods</copy>
	```

	![](images/k8-pods.png " ")

4. Continuously tailing the log of one of the pods

  $ kubectl logs -f <pod name>
  Example kubectl logs -f todo-deployment-657895dd59-qd89j

5. For debugging deployment issues, you can run describe command and look at the errors at the end.

    kubectl describe pod <pod name>
    Example kubectl describe pod todo-deployment-657895dd59-qd89j

6. Now that your application has a external ipaddress, you can now access it both through curl and any web browser.
    ```
    curl -X GET http://<external_ipaddress>:8080/todolist
    or
    open a browser to the link http://<external_ipaddress:8080/todolist.
    ```
    ![](./images/Application.png " ")

    In this app,  Flask app provides end points and data from ADB. The GUI is rendered using https://github.com/Semantic-Org/Semantic-UI in a html templates.

## Task 6: Configure the API Gateway (optional steps)

A common requirement is to build an API endpoints for docker applications with the HTTP/HTTPS URL of a back-end service.
This can be done using Oracle API Gateway service.

The API Gateway protects any RESTful service running on Container Engine for Kubernetes, Compute, or other endpoints through policy enforcement, metrics and logging.
Rather than exposing the Todo App directly, we will use the API Gateway to define cross-origin resource sharing (CORS).

1. From the hamburger  menu navigate **Developer Services** > **API Management > Create Gateway**
   ![](images/API-Gateway-menu.png " ")

2. Configure the basic info: name, compartment, VCN and Subnet
    - VCN: pick on of the vitual circuit network
    - Subnet pick the public subnet (svclbsubnet)

	The click "Create".
  	![](images/Basic-gateway.png " ")

3. Click on Todolist gateway
    ![](images/Gateway.png " ")

4. Click on Deployments
   ![](images/Deployment-menu.png " ")

5. Create a todolist deployment
   ![](images/basicInformationdeploment.png " ")

6. Configure the routes: we will define two routes:
    - /tododo for the first two APIs: GET, POST and OPTIONS
    ![](images/Route-1.png " ")

    - add  /todolist/delete route API: (GET, PUT and DELETE)
	   ![](images/Route-2.png " ")

     - add  /todolist/add route APIs.
 	   ![](images/Route-3.png " ")

     - add  /todolist/update route API.
      ![](images/Route-4.png " ")




## Task 7: Testing the backend application through the API Gateway

1. Navigate to the newly create Gateway Deployment Detail an copy the endpoint
   ![](images/Gateway-endpoint.png " ")

2. Testing through the API Gateway endpoint
  postfix the gateway endpoint with "/todolist/todos" as shown in the image below


  It should display the Todo Item(s) in the TodoItem table. At least the row you have created in Lab 1.

   ![](images/withGateway.png " ")

That is it. You have now exposed the applications endpoints through Oracle API Gateway.

## Acknowledgements

* **Author** -  - Vijay Balebail, Dir. Product Management.
