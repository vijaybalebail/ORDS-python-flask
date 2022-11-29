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

		SELECT r.privilege_id, r.privilege_name, r.role_id, r.role_name, m.pattern, ct.client_name
		FROM   user_ords_privilege_roles r, user_ords_privilege_mappings m, user_ords_client_roles ct
		where  r.privilege_id=m.privilege_id
		and ct.role_name = r.role_name;
  </copy>
  ```


## Task 5: Get bearer token using client and secret.
1. Navigate to the rest console and click on the security tab. From the drowdown, choose OAuth Client.

	![](images/oauthclient.png " ")

2. Choose "Get Bearer Token" from the upper right corner of the ords_client.

   ![](images/oauthtoken1.png " ")


3. Choose Bash and then copy the cURL scrip and run it on cloudshell.

   	![](images/oauthtoken2.png " ")

4. Choose "Get Bearer Token" from the upper right corner of the ords_client.

   ![](images/oauthtoken1.png " ")

5.  Get the Bearer Token after running the curl script in cloud shell.
  Example.
	```
		vijay_bale@cloudshell:~ (us-ashburn-1)$ curl --user 8nEWrUbjYz6Ds_f2Zn1F0w..:1zSOCPLPECkPGX67bKWPMA.. --data 'grant_type=client_credentials' https://gxkzxwrxbhfdowb-oauthords.adb.us-ashburn-1.oraclecloudapps.com/ords/admin/oauth/token
		{"access_token":"AgkfCYvqSBdGKLnLFksLtA","token_type":"bearer","expires_in":3600}vijay_bale@cloudshell:~ (us-ashburn-1)$
vijay_bale@cloudshell:~ (us-ashburn-1)$
```
 6. Now pass the token as header to the ORDS URL using cURL.
     Example.


		 ```
		  curl  -i -k -H"Authorization: Bearer AgkfCYvqSBdGKLnLFksLtA"  'https://gxkzxwrxbhfdowb-oauthords.adb.us-ashburn-1.oraclecloudapps.com/ords/admin/rest-fullstack/rc1/49/2500000/7'

		 ```

		   The result should looke similar to what we got in the previous lab without OAuth.

		 ```
			 vijay_bale@cloudshell:~ (us-ashburn-1)$  curl  -i -k -H"Authorization: Bearer AgkfCYvqSBdGKLnLFksLtA"  'https://gxkzxwrxbhfdowb-oauthords.adb.us-ashburn-1.oraclecloudapps.com/ords/admin/rest-fullstack/rc1/49/2500000/7'

				HTTP/1.1 200 OK
				Date: Thu, 24 Nov 2022 00:06:56 GMT
				Content-Type: application/json
				Transfer-Encoding: chunked
				Connection: keep-alive
				ETag: "ailFRZTmWNh4VgnQNt8/hXxK47bl5THxmPQmDacNHHA50rNlvLGVrhmdBE7GC0orkQwfkv/GijVF2AVZszsvIw=="

				{"items":[{"age":50,"balance":2675000},{"age":51,"balance":2862250},{"age":52,"balance":3062608},{"age":53,"balance":3276991},{"age":54,"balance":3506380},{"age":55,"balance":3751827},{"age":56,"balance":4014455},{"age":57,"balance":4295467},{"age":58,"balance":4596150},{"age":59,"balance":4917881},{"age":60,"balance":5262133},{"age":61,"balance":5630482},{"age":62,"balance":6024616},{"age":63,"balance":6446339},{"age":64,"balance":6897583},{"age":65,"balance":7380414},{"age":66,"balance":7897043},{"age":67,"balance":8449836}],"hasMore":false,"limit":1000,"offset":0,"count":18,"links":[{"rel":"self","href":"https://gxkzxwrxbhfdowb-oauthords.adb.us-ashburn-1.oraclecloudapps.com/ords/admin/rest-fullstack/rc1/49/2500000/7"},{"rel":"describedby","href":"https://gxkzxwrxbhfdowb-oauthords.adb.us-ashburn-1.oraclecloudapps.com/ords/admin/metadata-catalog/rest-fullstack/rc1/49/2500000/item"},{"rel":"first","href":"https://gxkzxwrxbhfdowb-oauthords.adb.us-ashburn-1.oraclecloudapps.com/ords/admin/rest-fullstack/rc1/49/2500000/7"}]}vijay_bale@cloudshell:~ (us-ashburn-1)$
			```
 We have now successfully verified that the ORDS works with Oauth. Now in the next lab, we can demonstrate how we can get the same token from within a python application.

## Acknowledgements

* **Author** -  - Vijay Balebail, Dir. Product Management.
