# How to oAuth toolkit

## What is oAuth toolkit?

oAuth toolkit is a python package that extends  [oauthlib](https://github.com/oauthlib/oauthlib) for Django framework. It contains basic implementation of oAuth1, oAuth2 and OpenID connect protocols. 

## What is in it?

- A complete set of rest apis for authorization, token exchange, revoke token, token introspect,
- Application management
- Scope management
- Remote user verification
- OpenID support
- OAuth2 flows [authorization code, client credentials, legacy password, hybrid]
- DRF support

## Where to start?

If you are new to oauth toolkit try out the [tutorial](https://django-oauth-toolkit.readthedocs.io/en/latest/tutorial/tutorial.html), after that lets start with planning out for project structure. After you know what are your targets explore `oauth_provider.models.py` and figure out if existing models suit your needs. All models can be swapped, they can be inherited to a custom model class and used in place of existing one. 

The reason I'm starting with this is 

1. This is a 2 step process first the migrations are done and then changes in settings.py are applied. doing them in one go was not successful for me
2. After a successful swap, you might lose access to existing data, Data needs to be transferred separately

Also the documentation are show one example here is a sample code for Configuring all models

This is how models are configured in oauth toolkit

```python
# oauth_provider/settings.py
APPLICATION_MODEL = getattr(settings, "OAUTH2_PROVIDER_APPLICATION_MODEL", "oauth2_provider.Application")
ACCESS_TOKEN_MODEL = getattr(settings, "OAUTH2_PROVIDER_APPLICATION_MODEL", "oauth2_provider.AccessToken")
ID_TOKEN_MODEL = getattr(settings, "OAUTH2_PROVIDER_ID_TOKEN_MODEL", "oauth2_provider.IDToken")
GRANT_MODEL = getattr(settings, "OAUTH2_PROVIDER_GRANT_MODEL", "oauth2_provider.Grant")
REFRESH_TOKEN_MODEL = getattr(settings, "OAUTH2_PROVIDER_REFRESH_TOKEN_MODEL", "oauth2_provider.RefreshToken")
```

This how models can be reconfigured, make sure new models inherit class with 'Abstract' prefix ([official example](https://django-oauth-toolkit.readthedocs.io/en/latest/advanced_topics.html#extending-the-application-model)) 

```python
# customapp/settings.py
OAUTH2_PROVIDER_APPLICATION_MODEL = "customapp.Application"
OAUTH2_PROVIDER_APPLICATION_MODEL = "customapp.AccessToken"
OAUTH2_PROVIDER_ID_TOKEN_MODEL = "customapp.IDToken"
OAUTH2_PROVIDER_GRANT_MODEL = "customapp.Grant"
OAUTH2_PROVIDER_REFRESH_TOKEN_MODEL = "customapp.RefreshToken"
```

> Note: Try not to import a class directly from oauth_provider instead look for a get_XX_class method at the bottom of respective file. Reason being that in future if there is any class that needs to be overridden or replaced you will not have to go to the depth of project and rename all imports.
>
> ```python
> # if you need to use application class 
> from oauth_provider.models import Application
> # try this instead
> from oauth_provider.models import get_application_model
> Application = get_application_model()
> ```



## What next?

Next should be integrating oauth urls in your url configuration. I recommend to include oauth_provider.urls in root url.py of the project. The default url configuration will contain Application and token management pages. If they are not required they can be removed as follows

```python
# rootapp/urls.py
from django.contrib import admin
from django.urls import include, path
from oauth2_provider.urls import base_urlpatterns, oidc_urlpatterns, app_name as oauth_appname


urlpatterns = [
    path('admin/', admin.site.urls),
    path('o/', include((base_urlpatterns + oidc_urlpatterns, oauth_appname), namespace=oauth_appname)),
]
```

The special thing about this is that namespace of oauth_toolkit will remain intact, that means the reverse will work properly and internal library reverse calls will not break. It looks tempting to put this in some app other than root, that will cause a conflict in namespace and may break something inside oauth toolkit.

## The Auth Flows

Oauth toolkit supports 4 authorization flows

1. Authorization Code flow
2. Client Credential flow
3. Legacy Password flow
4. Hybrid flow

### Authorization  Code flow

The most commonly used flow, that goes like this authentication --> token --> access token with some url redirects. Access Token represents a user

### Client Credential flow

This is for inter service communication, an app can authenticate with `client_id` and `client_secret` . Used in resource server for introspection. Token generated by this does not represent a user.

### Legacy Password flow

This is depreciated it is a basic flow that exchanges user credentials and token in one transaction. This flow is not recommend but it will work for monolith application

### Hybrid flow

This is for a case where backend and frontend need 2 separate tokens, it is same as Authorization Code flow but with an  extra token exchange for frontend. Before working on this make sure you know about URL [fragment](https://en.wikipedia.org/wiki/URI_fragment).

## The Overrides

An override is something needed to add/remove features from python class. For oath toolkit there are many methods that are intentionally left empty, so that developers can import, inherit in their class and add additional features according to their requirements. This is optional but **SettingsScopes** and **OAuth2Validator** two classes that need some tweaks for making additional settings work properly.

### SettingsScopes

```python
# oauth_provider.scopes.py
from .settings import oauth2_settings # see oauth_provider.settings.py for default SCOPE, _SCOPE and _DEFAULT_SCOPES values

class SettingsScopes(BaseScopes):
    def get_all_scopes(self):
        return oauth2_settings.SCOPES

    def get_available_scopes(self, application=None, request=None, *args, **kwargs):
        return oauth2_settings._SCOPES

    def get_default_scopes(self, application=None, request=None, *args, **kwargs):
        return oauth2_settings._DEFAULT_SCOPES
```

My findings after try to implement custom scopes is that it is a tiny mess, since oauth toolkit in a wrapper around oauthlib The control of Scopes is distributed between two libraries. By default there are two scopes read and write configured in the settings, anyone who asks for them gets them. This is not suitable if the project consists of multiple types of users or multiple levels of access. Before stating a solution lets understand what the three methods do.

#### get_all_scopes

This methods is being used at a number of places in oauth toolkit for getting all scopes that the project supports, *by all it means all*. Authorization does not work if request has some scope that this method is not returning.

#### get_available_scopes

This method is special as it has 2 arguments **application** and **request** which gives us the opportunity to trim down scopes based on application or user in request. But the limitation here is that the built in usage of this method is either by application or by request. So extended logic should be written with that in mind.

#### get_default_scopes

*404* usage not found ???? , it is safe to just skip this method.

Example:

```python
from oauth_provider.scopes import SettingsScopes # since class is being override cannot use get method
from oauth_provider.models import get_application_model
class CustomSettingsScopes(SettingsScopes):
    # Add Scopes dynamically
    def get_all_scopes(self):
        scopes = oauth2_settings.SCOPES
        applications = list(get_application_model().objects.all().value_list('name', flat=True))
        temp = " application access"
       	additional_scopes = {k: k+temp for k in applications}
        scopes.update(additional_scopes)
        scopes['admin'] = "For admin access"
        return scopes 

    def get_available_scopes(self, application=None, request=None, *args, **kwargs):
        if request and request.user and request.user.is_superuser:
            return oauth2_settings._SCOPES + ['admin']
        return oauth2_settings._SCOPES
```
```python
# root/settings.py
OAUTH2_PROVIDER = {
    "SCOPES_BACKEND_CLASS" : "custom_app.override.CustomSettingsScopes",
    ....
}
```
#### Scopes that are available but not active

```
openid : For openid connect, requires application to have Encryption method set 
introspect : A token can only be introspected if this scope is available in token
profile : To return extra claims in id_token and userinfo endpoint
email : To return email
phone : To return phone number

```



### OAuth2Validator

This class is loaded with methods that validate a number of things and it is really easy to break the whole of oauth toolkit but there is a method that needs to be configured to add additional information in claims

```python
from oauth_provider.oauth2_validator import OAuth2Validator
class CustomValidator(OAuth2Validator):
	def get_additional_claims(self, request):
        if not request:
            return {}
        # For these claims to appear appropriate scopes should be in the token
        return {
            "family_name": request.user.first_name,
        	"given_name": request.user.last_name,
            "email": request.user.email,
            "picture": reqeust.user.profile_pic,
        }
```

```python
# root/settings.py
OAUTH2_PROVIDER = {
    "OAUTH2_VALIDATOR_CLASS": "custom_app.override.CustomValidator",
    ....
}
```



## The OpenID connect support

[OpenID](https://openid.net/connect/) is simply an encoded token( not access token) that contains information about user, some timestamps and it can be verified via public keys from **jwks endpoint**. 

## The Introspect and its limitation

Introspect is a feature that enables remote server(with token / credentials) to validate incoming access tokens. This seems a bit redundant as each time a resource is requested server will make a introspection request. But it is not like that oauth toolkit only introspects if token is already not present in its database otherwise it will request introspection and save token locally for future use until it expires. This creates a problem, if a token gets revoked before expiry there is not way of revoking token from remote server. This is where expiry times of access token and refresh token come to play. If access token expires in next 15 minutes or 30 minutes resource will automatically invalidate token and refresh token needs to be more than 30 minutes in order to continuously exchange new tokens after expiry on back. 

Other method is to create an API interface for each resource server that will delete targeted tokens 

### How to setup introspection in Resource Server

In django project install oauth toolkit. Acquire introspect url and application token/credentials(client_id,client_secret). Place them in settings.py like this

``` python
OAUTH2_PROVIDER = {
    ...
    'RESOURCE_SERVER_INTROSPECTION_URL': 'https://example.org/o/introspect/',
    'RESOURCE_SERVER_AUTH_TOKEN': '3yUqsWtwKYKHnfivFcJu', # OR this but not both:
    # 'RESOURCE_SERVER_INTROSPECTION_CREDENTIALS': ('rs_client_id','rs_client_secret'),
    ...
}
```
An additional perk of using django with oauth toolkit is that a partial user object will be created on first token access in the resource server. In a Resource server models can be linked to User model via foreign key and everything will work with any additional setup ???????????