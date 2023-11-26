## Google Open ID

Google OAuth2 API can be used for authorization and authentication, since it conforms to the [OpenID Connect](https://openid.net/connect/) specification, and is [OpenID Certified](https://openid.net/certification/).

OpenID Connect specification set has a [OpenID Connedt Doscovery](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata) specification
which allows an application to retrieve the configuration of the OpenID Connect compliant Id provider.

The url to retrive [OpenID Provider Metadata for Google IP](https://developers.google.com/identity/protocols/oauth2/openid-connect#discovery)
is `https://accounts.google.com/.well-known/openid-configuration`.
We can get URL of the endpoints, supported clams, aupported scopes and other configuration from  the information exposed from this endpoint.

This tutorial

### Set Up

Here you will find the short version of the OAuth2 set up described in the [Google tutorial](https://developers.google.com/identity/protocols/oauth2/openid-connect#appsetup)

#### Create a new OAuth client

We will create a java web application which we will test on the local machine.
In this case it is a private client for which we must obtain a `client_id`and `client_secret`.

To add a new OAuth2 client for Web application follow the steps:
1. Go to [Google developer's console, Credentials](https://console.developers.google.com/apis/credentials)
2. Add a new client of "Web Application" type.
3. Fill in "Name" field with any name you like
4. Fill in Authorized redirect URIs" with `http://localhost:8080`

#### Consent screen

Google require to cutomize a Consent screen corresponding to a project which is shown to the user each time she logs in to your application.
![Concent screen explanation(Google Consent screen page)](3-consent-google-explanation.png)

To fill in a consent screen, go to  [Consent Screen](https://console.developers.google.com/apis/credentials/consent) and fill in the required information.
You can find more details in [here](https://developers.google.com/identity/protocols/oauth2/openid-connect#consentpageexperience) in the Google tutorial.

You application should have "Testing" status to avoid application verification and public access to the application.
 
### Project structure and dependencies

Project consists of 2 modules:
* web-app - the spring boot application
* front - Angular application

#### Web app application







https://developers.google.com/identity/protocols/oauth2/openid-connect#discovery

# Resources
* https://www.baeldung.com/spring-security-openid-connect
* https://developers.google.com/identity/protocols/oauth2/openid-connect


