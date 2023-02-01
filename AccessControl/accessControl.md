NOTE! in order to access the secured data via `https://scorpio.staging.odala.kiel.de/ngsi-ld/v1/entities` you need to present addional header and value: `x-admin-access: allow`

NOTE! This solution will be depecrated with APISIX and Keycloak based on solution at future time point.


### Access control layer for ODALA

### 1. Introduction

Components we utilise are [API Umbrella]() and [Keyrock](). Umbrella acts as a PEP and Keyrock as PDP. In sort, when an HTTP request with Bearer token arrives to the system, Umbrella receives it. Umbrella evaluates Sub-rules, and if there is a match, it will pass the Bearer token for Keyrock for introspection. Keyrock then evaluates the request based on Roles and returns a decision to allow or deny request to Umbrella. Umbrella then enforces this decision.

## 1.1. How does the system work in detail.

Umbrella together with Keyrock allows fairly flexible possibilities how to build access control rules. Here is a sequence with details. Prerequisite is that the system is configured before.

1) Request arrives to umbrella. Umbrella checks if the URL matches any API endpoints defined (frontend host + frontend prefix):

![](img/u1.PNG)

2) If there is a match, then the Global Request Setting are checked:

![](img/u2.PNG)

Since the role defined in the Global Request Setting is something no-one should have..

3)..Umbrella proceeds to check the Sub-URL rules:

![](img/u3.PNG)

In the rules order matters. It is important to know that the first matching rule is processed and the subsequent ones are NOT evaluated.

4) In the Sub-URL rule the Method, Regex, Required headers, Authorization Mode, Required Roles and Override check are evaluated:

![](img/u4.PNG)

As stated before, these rules need to be unique to avoid incorrect decisions. We have now opted to use few simple rules, but this can be changed in the future, as the framework is quite flexible. The only head ache is to define correct Regexes on both Umbrella and Keyrock.

When the Sub-URL rule is matched, request is made to Keyrock. 

5) Keyrock checks the user belonging to the Application and having the role defined in Umbrella matched rule:

![](img/k1.PNG)

If this is all good, request is passed on. If not the request is denied.

### 2. Example Keyrock configuration

Here is an example of the Keyrock configuration and steps to follow:

Login to the Keyrock using Keyrock Admin credentials (Credential location depending on the system; In Kiel they are stored in Gitlab) to login via 'https://accounts.staging.odala.kiel.de'. The admin user is automatically created via CI/CD pipeline and the creadentials are as secrets in the deployment.

TBD

This is the current state. We have opted not to automate any of the steps, as we are looking how to leverage i4Trust and Data Spaces as access control layer. This would most likely introduce significant changes, and we prefer to avoid double work.

### 3. Example Umbrella configuration

Umbrella is a reverse proxy and acts as an API management conponent. In this section we give an example how to configure Scorpio broker API and set access controls. It also serves as a training, so you understand better what is happening in your system, and how it is configured or how it can be configured.

Before proceeding, it is necessary to complete Keyrock configuration and we recommend to read through this whole document.

Please note that this is an example configuration only; you need to adapt this to match you deployment.

1. Go with the browser to the URL; In case of Kiel it is 'https://umbrella.staging.odala.kiel.de/admin/login' Use the Umbrella Admin credentials (Credential location depending on the system; In Kiel they are stored in Gitlab) to login. With the current deployment model, the Umbrella Admin username and password is created first time user logs in to the system. This is described in [Umbrella documentation](https://api-umbrella.readthedocs.io/en/latest/getting-started.html#login-to-the-web-admin). 

2. On the top menu, click 'Configuration". Click "Add and API backend". After this you should see the following screen:

![](img/ua1.PNG)

3. Start filling the information. Let's start with 'name' and 'Backend protocol':

![](img/ua2.PNG)

4. Next up is adding a server. This server is the service that will receive the traffic Umbrella proxies, when this API un configured. Click 'Add Server' and fill in the details:

![](img/ua3.PNG)

The server name is the URL of the Service your Kubernetes cluster has assigned for the service you are trying to connect. If you do not know what to enter here, check this from the Kubernetes cluster, or ask this from who ever is responsible for the Kubernetes environment. In our case the service is the Scorpio NGSI-LD broker by NEC.

5. 'Frontend host' is needed next. This is the URL domain part the service will be accessible from:

![](img/ua4.PNG)

6. Then we need the 'Backend host'. This is the same URL as set in the step 4.

![](img/ua5.PNG)

7. So far we have the servers and front end configured. Now we need to define the endpoint part the service is running in:

![](img/ua6.PNG)

After this step is completed, we have configured the basics for the API. Next we need to define the rules for the Access Control. In this section we need details from Keyrock, so please login with Admin user to your Keyrock.

8. Expand the 'Global Request Settings' section. Fill the information accordingly:

![](img/ua7.PNG)

Brief explanation on the values needed. The 'Allow External Authorization' tick box means that we are connecting Umbrella with external Component for Authorization. This component is in our case Keyrock. In order to connect, we need few other details. To connect to the correct Application in Keyrock we need to enter the 'IDP App ID' from Keyrock's application. For this you need to login to Keyrock with Admin account and take the value from the UI. If you are not sure, check the previous section where Keyrock was configured.

Then we need to tell that we are dealing with 'Role based' (the 'R' in RBAC) authorization mode. Finally give a random value to 'Required Roles'.

> Note on the Required roles: This needs to be a role no one has in Keyrock. This ensures that the Global access controls are never applied, and the defined 'sub-URL' settings (defined later) as applied. We recommend that you use random string as a value. The control is bit finicky, make sure the value is added.

If you were wondering, how does Umbrella know where Keyrock is, that is a very good question! This is (again in our setup) defined in the Gitlab's Umbrella project 'resource.yml' file.

9. Almost there! Next is the last part of the API configuration, 'sub-URL settings'. Expand the selector and click 'Add URL Settings':

![](img/ua8.PNG)

Next fill in the details:

![](img/ua9.PNG)

Brief explanation on the values: 'HTTP method' defines the allowed method for the rule. 'GET' effectively means that you can read something from the Service. 'Regex' this value is matched when the request comes in. '^/' translates to 'everything until "/"'. 'Authorization Mode' needs to be left to 'Default: Role based'. 

Then comes the interesting part. 'Required Roles'. This is a role you (hopefully) have defined in Keyrock. Enter it excatly like in Keyrock. 

Lastly, the 'Override required roles from "Global Request Settings"' needs to be checked. This will do as described on the label, if makes the role to take effect. 

> Putting all these things together allow the following query to be executed: The service with defined domain (scorpio.staging.odala.kiel.de) concatenated with the end point (/ngsi-ld/v1/) may be accessed via GET request accompanied by Bearer token belonging to a Keyrock user who has the Required Role. Granted, that sentence is bit convoluted, but readit few times. Once you got that, then you got what this all means.

10. Save and publish:

Hit the 'Save' 

![](img/ua10.PNG) ![](img/ua11.PNG)

and publish:

![](img/ua12.PNG) ![](img/ua13.PNG)

11. Test the configuration

It is a good idea to check if all works as expected. For that refer to the next section. Please keep in mind that in order to have a functioning configuration, both Keyrock and Umbrella configuration need to be done!

That's it! If you have any questions, please do not hesitate to open an Issue in Gitlab, or if that is not an option, contact ilari.mikkonen@contrasec.fi

This is the current state. We have opted not to automate any of the steps, as we are looking how to leverage i4Trust and Data Spaces as access control layer. This would most likely introduce significant changes, and we prefer to avoid double work.


### 4. End user how to

Take this [postman collection](Postman/ODALA-Kiel.postman_collectionv2.json) it's your friend.

[Sign up](https://accounts.staging.odala.kiel.de/sign_up/) for Keyrock account: 

![](img/e1.PNG)

Notify the admin.

In order to write(POST, DELTE + other HTTP verbs) or read(GET) data from the system you need to have a role given to you and you need to be added to an application. System admin needs to do that so make sure this has happened before proceeding.

1) You need to have the following detiails available: email for Keyrock user (-the one you just created), password for Keyrock user, client id and cliet secret from Keyrock. Email and Password you should have. Once notified by admin, login to Keyrock. You should see this application:

![](img/e2.PNG)

Open and reveal the Oauth2 secrets. 

![](img/e3.PNG)

Take all this information and plug it into the Postman collection variables. Get the Bearer token and update the bearer token in the variables: 

![](img/e4.PNG)

and execute the query:

![](img/e5.PNG)

If you have any problem with these steps, contact ilari.mikkonen@contrasec.fi we'll iterate.

### 5. Admin how to

When a new user signs-up to the system, enable the user account:

![](img/a1.PNG)

Then authorize the user into the application by using the search (username) and add by clicking the + icon

![image-1.png](img/image-1.png)

and then authorizing in the application

![](img/a2.PNG)

Search for the username and hit the "+" icon.

![](img/a3.PNG)

and give the user a desired role. More on the roles and definitions on the "Example Keyrock configuration" section.

![](img/a4.PNG)

Changes take effect immediate.

### 6. Links and other material


Copyright (c) Contrasec 2022
