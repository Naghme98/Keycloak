# Identity and Access Management part 1

## Task 1: Preparation

- Setup Keycloak
    - Install keycloak using docker image "https://www.keycloak.org/getting-started/getting-started-docker"
        - To run the container use

```
docker run -p 8081:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:17.0.1 start-dev -Dkeycloak.profile.feature.upload_scripts=enabled
```
 
- Make hostname sso point to 127.0.0.1, that will be used to access keycloak services (basicaly it is aliasing localhost, IP stays the same but browsers will treat it as a separate domain - important for cookies)
- Access keycloak at sso:8081
- Create realm as_lab
- Create sample user in this realm

- Launch sample app - installation instructions inside iam-lab-part1.zip

-----------------


### Implementation:


- Change the /etc/hosts:

![](https://i.imgur.com/HPOt5CO.png)

- Add realm:


![](https://i.imgur.com/e6iISTR.png)
    
- Add user:

    
![](https://i.imgur.com/qZNhNjq.png)
    
- Add password for user:
    
![](https://i.imgur.com/8BLkmAB.png)
    
- Lunch App:
- 
![](https://i.imgur.com/HmdPSDq.png)
  
    
    
----------------

## Task 2: Authentication and Single-Sign On

For this task you will use 2 parts of the app:

- "app-pokemon" - browser-side single page application that displays pokemon data.
    - It is located at http://localhost:3068/pokemon/:pokemon-name
- "app-trainer" - browser-side single page application that displays trainer pokemon decks.
    - It is located at http://localhost:3068/trainer

In this task you need to achieve Single-Sign-On support for those 2 apps.

Applications are already pre-configured to work with Keycloak,
but you need to deploy configuration on the Keycloak side.

Applications are using Keycloak JS adapter that follows OpenID Connect protocol.

Keycloak configs on the app side are located inside public directory (you do not need to edit them):
keycloak-pokemon.json and keycloak-trainer.json


### Task 2.1: Before you begin you must get familiar with terminology and concepts of OAuth 2.0 and OpenID Connect.

- a) What is client?
- b) What client types exist?
- c) What authentication flows exist?
- d) Why is Authorization Code Flow preferred over the others?
- e) What is Single-Sign On (SSO)?
- f) What is the difference between OAuth 2.0 and OpenID Connect?

--------------

#### Answers:

- a) Client is the application, service which wants to gain access to some resources. Like in our example, pokemon-app is the client that asks resource owner (us) to approve that it can have access. Then the rights and grants will be given through OpenID provider which is Keycloak in our example.
- b) based on OAuth rfc, there are 2 types of clients. "Confidential" and "Public". Their difference is that the first one can securely authenticate on the authorization server. For example web browser-based applications are "public" type cause everyone is using it and probably there is no such a need of security.
- c) The following ones exists:
    - Authorization Code Flow: for browser-based applications that redirect to the OP(openid provider- keycloak) and use access tokens. 
    - Implicit Flow: Similar to previous one but with less steps (reciving code step is omitted)

There are some grant methods too (There was no flow word for them but I saw them in serveral sites and I guess they are authentication flows too): 
    - Client credentials grant 
    - Resource owner password credentials grant (Direct Access Grants)
    - Device authorization grant
    - Client initiated backchannel authentication grant

- d) 
    - 1. Most of the applications are browser-based and this one is convineniant for web-based applications (it uses a simple redirect and no need for user to send his credentials to this client)
    - 2. Access token will go to the web server of the application not to the browser of the user which will reduce the expouser chance.

- e) It is an authentication scheme that let us login to several systems/services using one ID without any need to sign up to each of them. It is based on trust relation between services. For example now adays, we use signin with google. Most of the services trust google and when we login to it, they will automatically create a acount for us without forcing us to pass the signup procedure for their service.
- f) First OAuth was created. It is an authorization protocol (or some sites said framework) that contol authorization to a protected resources. OpenID is authentication protocol that will be used on top of the OAuth2.0. OAuth could be used in external partner sites to allow access to protected data without them having to re-authenticate a user.

-------------

### Task 2.2: Setup authentication

Сreate corresponding client for app-pokemon and show that you login is successful.
Then create corresponding client for app-trainer and show that SSO works.

--------------

#### Implementation:

- Add client for pokemon:

    
![](https://i.imgur.com/442jDjj.png)
    

- Enable the authorization for this client:
 
    
![](https://i.imgur.com/J31Tz3p.png)
    

After that I checked that to see if it was the case you wanted. I had the following lines for the features I enabled previously:
 
    
![](https://i.imgur.com/coBsGko.png)
    

And then I looked through the "keycloak-pokemon.json" file and I understood the accesstype should be public.


![](https://i.imgur.com/Tj8dNwT.png)


- Add client for Trainer:


![](https://i.imgur.com/EUGEA5Z.png)
    

After that if we want to login, it would redirect us to the keycloak and we can successfully login:


![](https://i.imgur.com/fA5nxmM.png)

And for the second site, cause I already logged in in keycloak, it didn't ask me to login there and got logged in trainer site automatically.


![](https://i.imgur.com/dHWq9Kd.png)
    
---------------

### Task 3: Analyze SSO workflow

Take a look at full-login process and auto-login process in the network tab of browser devtools and describe what is happening. Use “Preserve log” to prevent resetting on redirects

- a) What authentication flow is used?
    b) What is the difference between id_token and access_token?
    c) What is purpose of refresh_token?
    d) How is the session persisted? What will happen if you use incognito mode or another browser?
-------------------------

#### Answers:

- a) Authorization Code Flow
- b) id_token has the identity information about the user but access_token is a token that shows this client can access authorized resources without other authentications. The expiration time of access_token is shorted than the other ones and has different informations needed to indentify the grants granted to this client is encoded inside it.
- c) These are added to increase the user experience satisfaction. They are there to obtain new access_token cause as I told before, access_tokens will expire after a short time and with theis token, without any need for login, client can get new access_tokens. After refresh_token expriration, user should login. 
- d) As you could see keycloak sets a cookie for `sso` domain that contains current user session. When app will redirect again to the login page, keycloak will not prompt user for credentials and it can just use information about session associated with that cookie

#### Implementation:

First there is a GET request to the keycloak(sso:8081) that is the OpenID Provider. 
    
![](https://i.imgur.com/EbLE9sn.png)
    

- client_id: The id we set for our client (the website)
- redirect_uri: Where the response should be sent back.
- state: A Opaque value for maintaning state between request and callback
- response_mode = fragment: It shows authorization Response parameters should be encoded in the URI Fragment Identifiers added to the redirect_uri when redirecting back to the OAuth Client.
- response_type = code: It shows authorization code workflow 
- scope = openid: Shows it is a request for OpenID authentication and ID token.

Then I used the credentials for my user and logged in so:

In the following post request, my user's usename and password sent to the keycloak and in response an authorisation code redirected to the address that was indicated in "redirect_uri".


![](https://i.imgur.com/OTlhwpZ.png)
    
    
Also, A cookie named "KEYCLOAK_IDENTITY" was set that is for ID of the current user. The other cookies are for internal use of keycloak.

p.s: The state parameter should be the same as value for that was set in the previous request.

Next step is to retrive ID token from keycloak.
The code that we recived has no meaning for client and client will use it for reciving ID token. 

So there is a POST request:

![](https://i.imgur.com/gBgBOv9.png)
    
Inside the Request tab we can see that client sent the "code" that was received, grant_type which is "authorization code" type, client_id and redirect_uri.


![](https://i.imgur.com/6QlziFt.png)
    
    
In response, keycloak will send us "id-token", "access_token" and "refresh_token".


![](https://i.imgur.com/zIxVAlK.png)
    

That "token_type : 'Bearer'" is for ensusing that token reponse is compatible with "OAuth 2.0".


Now that one of the sites successfully logged in, lets see the request flow when we try to login in the other application and if there is any difference or not.


The first request is like the pervious state but in response, without redirecting to the keycloak, it sends the code for us:


![](https://i.imgur.com/lbDc3Bg.png)
    

And in the next step, client send a request to retrive the token:


![](https://i.imgur.com/2eNKRaw.png)

![](https://i.imgur.com/88ul2vT.png)

![](https://i.imgur.com/FkMO5yy.png)

       
     
----------------

## Task 4:  Resource access with role-based authorization
For this task you will use 2 parts of the app:

- "app-pokemon" - browser-side single page application that displays pokemon data.
    - It is located at http://localhost:3068/pokemon/:pokemon-name
- "api-pokemon" - API that provides pokemon data.
    - GET http://localhost:3068/api/pokemon/:name - get pokemon data - only users with readonly role allowed.
    - PUT http://localhost:3068/api/pokemon/:name - set pokemon date - only users with editor role allowed.


### Task 4.1: Enabling authorization

 Keycloak issues access tokens in JWT format. The advantage of JWT token is that all information that is required to make a decision about authorization can be extracted and verified from it.
 
a) What is the format of JWT token? How can you decode it, is there any standard tools?
b) How is JWT token issued and verified?
c) What is claim and what is scope? 
d) What is aud claim?

-----------------------------

#### Answers:

- a) It has three parts : "header", "Payload", "Signature" and the first two ones will be base64Url encoded. Inside payload, the user's claims are places. All these 2 encoded strings using a secret key or certificate and the algorithm specified will sign and this signature will be placed in concatination of the other two encoded header and playload. The final format is like "HHHH.PPPP.SSSS". For decoding we need only to decode the base64url of header and payload and for verifying, we need the secret code to produce the singature and then compare.  As I know we can do this procedure using most of the programming languages (write codes for it) and jwt.io is another place we can check, debuge and decode our tokens.

    hash_value = hash([base64UrlEncode(header) + “.” + base64UrlEncode(payload)], secret-key)
    Signature = base64UrlEncode(hash_value)


- b) I think I kinda explained this one in the previous part.
- c) Claim is a pair key/value that shows an entity has that property (value). for example {"name" : "Naghme"} is a claim. Scope is a group of claims that shows what access privilages are being requested. For example "profile" scope is there to request access to the profile claims.
- d) Resource Servers that should accept the token ( reception of access-token) in our example it is "pokemon-api". Important to note that services that are not listed in `aud` claim should reject token

-----------------

### Task 4.2: Verification in the application


Look at how the api-pokemon makes authorization decision and list the critical claims that are required for positive decision.
File: server/verify-access.ts

#### Answer:

First it would get the "Authorization" part of the header and checkes if it is "Bearer" type or not.
So, that means we need to use "Bearer-only" type for the next part.

Then it would work on the passed jwt token and in the begining verify the token with algorithm "RS256" and if the "aud" part was "api-pokemon" or not.

So, "aud" is a necessary claim till now.
The last step is to check user's role that is "readonly" or "editor". The key for this value is "resource_access.api-pokemon.roles". That means we need to enable "roles" in our claims.

-----------------

### Task 4.3: Create corresponding clients and add roles

Make changes to Keycloak such that correct required claims are included into access token.
Show that app-pokemon successfully works.

Note that for app-pokemon you already have associated client in Keycloak. Also Keycloak provides a special bearer-only type of client - it is for services that do not initiate user login.

Also, for debugging purposes Keycloak’s Admin console provides Evaluation tool that shows you what is included inside access token.

For valid JWT verification do not forget to replace hardcoded cert (in server/verify-access.ts) with the cert of your realm.
:::

#### Implementation:

First creating another client "api-pokemon" and set the access type to "Bearer-only":

    
![](https://i.imgur.com/FkGx3Vc.png)
    
Then I should create 2 roles for this client.
    
![](https://i.imgur.com/R7Jdflj.png)
    

I assigned roles to my user:
    
![](https://i.imgur.com/dGm2LK1.png)


I took the certificate from realm settings and put it into the service.ts file:

    
![](https://i.imgur.com/SnVTMMx.png)
    
Then if I login to the site I have both readonly and editor access:


![](https://i.imgur.com/NbXK56G.png)
    

- a. What are realm roles and client roles in Keycloak?
    - Realm roles: The roles that are created or we create and will be applied to the all of the clients in that realm (All of the clients in that realm can map them to their users)
    - Client roles: The roles that can be mapped between the users of that specific client not all of them.

- b. What is a composite role?
    - A role that has morethan one role assosicated with.


- c. How does browser-side app app-pokemon understand that form input should be enabled/disabled?
    - Client app reads roles from access_token and provide UI on that basis




-----------------------

## Task 5: Issuing fine-grained access token

By default Keycloak includes all available user roles from all clients into the access token. Considering least privilege principal, that should be fixed.


### Task 5.1:

Create another test-client with some-role and add this role to the sample user. After that inspect access_token token for app-pokemon. Is new role automatically included into it?

------------------------------
#### Implementation:

- Create test client:

![](https://i.imgur.com/As6Gele.png)
    
- Create role for it:
    
![](https://i.imgur.com/DeC9Nx8.png)

- Add new role to user:

![](https://i.imgur.com/ZIXDMKO.png)

And if we check the access token now, we can see that all the roles that this user has is included:

![](https://i.imgur.com/vLqZ2HU.png)

---------------------

### Task 5.2: 

Learn how exactly user information that Keycloak possesses is mapped inside to access token. Make changes to Keycloak such that for app-pokemon only api-pokemon roles are mapped to the access token, even if he has other roles.

a) How is claim added to access token?

Keycloak Admin Console provides scope evaluation tool for debugging purposes.

-----------

#### Implementation:

:::warning
How is claim added to access token?
:::

I create my mapper:

    
![](https://i.imgur.com/Wb1ato1.png)
    
And turned the default one off:

![](https://i.imgur.com/L9bXPVS.png)


Then it is the result:

    
![](https://i.imgur.com/Q09sjMs.png)

Lets see what happenes in our access-token now:

    
![](https://i.imgur.com/RlI9DwJ.png)

    
You can see that our action was successfull and now there is only the api-pokemon roles in our token.


Okay, I guess it was not what you wanted cause in this case the new mapper would be used for every client we have.
So, I create the below configuration for app-pokemon and the result was correct. Only the roles from api-pokemon was there in the access token.

![](https://i.imgur.com/vfGHkX4.png)


To test if I am right, I will create another one to include "some-role":

![](https://i.imgur.com/5fgR4t4.png)

Successfully added "some-role".

Don't be mad at me, but, when I went through the other part I thought I did this part wrong.

I could get what you want in this way:

![](https://i.imgur.com/ttzQzzH.png)

And thats it.
    
![](https://i.imgur.com/MZCvUGy.png)
    

-------------------

## Task 6: Custom scopes

Keycloak allows you to create your custom scopes to include additional information to access token, for example custom user attribute.

Make changes to Keycloak such that access token for issued for app-pokemon will include favorite_color user attribute.

-------------

#### Implementation:

First I add the attribute to my user:

![](https://i.imgur.com/8sm0J1u.png)

Then create my custom scope 

![](https://i.imgur.com/3w8ejjX.png)

You can see the value in the token.

![](https://i.imgur.com/xfRwnc3.png)
    

