![status](https://badgen.net/github/status/micromatch/micromatch)
# A Roadmap to oAuth toolkit in Django

This is a journal about my findings while try to make microservice project.

## Roadblock 01 

While planning out a basic structure for microservices there were roadblocks at every step, although I have developed a few monolith applications this did not go easy. One of the many roadblocks was authentication process, in a basic app it is either a session based authentication or a login rest api that exchanges user credentials with token and then token is used for rest of the processes. 

The requirement was something like this , there will be *x* number of mini projects (distributed over unknown systems) that will be for internal use and then there can be *y* number of 3rd party applications that can use some of the previously mentioned projects. Using traditional approach is acceptable if there are no 3rd party applications. The most basic of setup for authentication needs 2 api one for token exchange and second for verifying that token.

### Why OAuth2?

Because 3rd party will get access to a user credentials and this is not something that can be left on the good will of other side 😅. This is where oAuth2 comes in the story, oAuth2 is quite complex so read [this](https://auth0.com/intro-to-iam/what-is-oauth-2/). oAuth2 is not for **authentication**(*authN*) it is only for **authorization**(*authZ*) 

#### What's the difference?

Exchanging credentials and verifying a user is **authN** and transferring authority to another application(frontend, 3rd app) is **authZ**

oAuth2 allows us to separate Authentication process into two parts authN and authZ. This way we can make sure user credentials  (email, username, password) directly reach us without any intervention.

#### What's the process of oAuth2?

After a user is authenticated it receives a token lets call it **grant token**. This token is useless as it will not work for any resource. Its sole purpose is to exchange it with an **Access Token**. An access token is the one  that will have access to resource based on its *Scopes*.

#### What are Scopes

A Scope is a permission it is requested in the beginning of authorization process and is attached to the final *Access Token*. A user can ask for any of the available scopes but will only get what that user is allowed to. The restriction is implemented on the resource side. The resource gets an access token, along with scopes then decides whether it qualifies to use that resource 👮‍♂️. 

#### Are Scopes encoded in Access Token?

No, Access Token has no information in it, it is simply a piece of string. There are two cases, Auth Server and Resource Server.

##### Auth server

This project creates token while authorization process that is  why it has all the scopes saved so  when ever a token comes scopes are fetched from database

##### Resource Server

This project needs to verify token by using a feature call **Introspection**. On Auth server there is an api which takes token and verify's it and responds with scopes and other related data.

#### What about the user info?

In OAuth there is a concept of claims. Claims are just keywords that let you control amount of user info. [this](https://github.com/jazzband/django-oauth-toolkit/blob/f835a243811aa9fcb54f559350daf5758249c66b/oauth2_provider/oauth2_validators.py#L73) is a mapping of standard claims and their respective scopes, it means that a  claim will work only if  user has a specific scope.

#### How to logout?

In oAuth2 there is no such thing as logout, Just delete the token and user will lose access to resources😋 

This is what I would like to say but it is a little complicated, There is another token known as the **Refresh token**. It comes along with the *Access token*. OAuth proposes a system where access token has small duration for validity and refresh token has relatively large duration of validity. Users are supposed to use Refresh token to get new Access token whenever an Access token expires. By this logic if we remove refresh token on Auth server user will not be able to continue after Access token expires. To Invalidate/Remove Refresh token OAuth uses a term **Revoke**. The user will immediately lose access after using revoke api. In practice an Access Token can last for as low as 5 minutes and a Refresh Token can last for weeks or even months.

**Note:** Before going further oAuth protocol assumes that everything will be done on **HTTP protocol under SSL encryption** if any of the token gets leaked while a transaction the user will be compromised. Both Access Token and Refresh Token are supposed to be secured safely on client side.   

## Roadblock 02

At this point the user credential leakage issue is solved, We have an AUTH server that will grant a token and resource servers can verify that token. The next thing is service breakdown. How should you break a Single code base into multiple small code bases. There is a way where each mini project is directly accessed by external user. It requires an Api Gateway for management and access control. The other way is to split heavy lifting tasks as sub project. For example for sending email a separate project can be setup for sole purpose of sending Email, a mini project to run cleanup jobs periodically etc. Each split of project may or may not require extra resource in this case. Lets say a simple Django app requires A machine to run code and a database instance, if we split the email service we only need a new machine to run service, periodic jobs service can be connected to main database with read only restriction, another machine. But if there is a service that requires data storage number of resources will increase rapidly this is applicable on both cases.

## Sanity Rules

Rules to make sure that microservice do not become Spaghetti. To understand watch this [video](https://youtu.be/gfh-VCTwMw8)

1. Try your best to make a linear stack, avoid circular traps, implement circuit breakers and limited number of retries
2. Make following assumption during development
   1. The network will not be reliable
   2. Latency will never be zero
   3. Bandwidth will be limited
   4. The network will not be secure (network is public etc)
   5. Topology will change (domain names , sub domains , ip addresses etc)
   6. There will be multiple administrators (You are not the sole manager of the project)
   7. The network will be uneven/asymmetric for different parts of microservice

## How to oAuth toolkit?

### What is oAuth toolkit?

oAuth toolkit is a python package that extends  [oauthlib](https://github.com/oauthlib/oauthlib) for Django framework. It contains basic implementation of oAuth1, oAuth2 and OpenID connect protocols. 

Coninue [here](https://github.com/AbdulMoeed-140212/OAuthToolkit_Django_Roadmap/tree/main/how_to_oauthtoolkit)
