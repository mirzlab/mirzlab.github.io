---
layout: post
title: CAS 5 as an OpenID Connect provider
---

This post shows how to configure CAS 5.1.x as an OpenID Connect provider.

-----

## LDAP Configuration

To authenticate our users, we will need an LDAP server. I am using [this docker image](https://github.com/jtgasper3/docker-images/tree/master/389-ds) which provides a simple and ready to use OpenLDAP server.


```bash
git clone https://github.com/jtgasper3/docker-images.git
cd docker-images/389-ds
```

I edited the users.ldif as follow but you can edit it to fit your needs :

```bash
dn: uid=jdoe,ou=people,dc=example,dc=edu
objectClass: simpleSecurityObject
objectClass: organizationalRole
objectClass: inetorgperson
uid: jdoe
cn: John
sn: Doe
displayName: John Doe
employeeNumber: 123456789
userPassword: password
description: Demo user
```

Then build and run the container :

```bash
docker build --tag="jtgasper3/389ds-basic" .
docker run -d -p 10389:389 --name="ldap-server" jtgasper3/389ds-basic
```

## CAS Configuration

First retrieve the latest CAS overlay project :

```bash
git clone https://github.com/apereo/cas-overlay-template.git
cd cas-overlay-template
```

Edit the pom.xml and add the following dependencies :

```xml
<!-- Enabling OIDC support -->
<dependency>
  <groupId>org.apereo.cas</groupId>
  <artifactId>cas-server-support-oidc</artifactId>
  <version>${cas.version}</version>
</dependency>

<!-- Needed for OIDC client registration -->
<dependency>
  <groupId>org.apereo.cas</groupId>
  <artifactId>cas-server-support-json-service-registry</artifactId>
  <version>${cas.version}</version>
</dependency>

<!-- Support LDAP authentication -->
<dependency>
  <groupId>org.apereo.cas</groupId>
  <artifactId>cas-server-support-ldap</artifactId>
  <version>${cas.version}</version>
</dependency>
```

Create the following folders

```bash
# This folder will contain all CAS properties
mkdir /etc/cas/config

# This folder will contain CAS service definitions (OIDC clients)
mkdir /etc/cas/services

# Log folder
mkdir /etc/cas/logs
```

Copy all required files to the config directory :

```bash
cd cas-overlay-template
cp etc/cas/config/* /etc/cas/config
```

Here is the content of my /etc/cas/config/cas.properties file (of course you will need to edit it to fit your environment) :

```bash
# CAS server URL
cas.server.name: https://cas.example.com:8443
cas.server.prefix: https://cas.example.com:8443/cas

# Logging module configuration
logging.config: file:/etc/cas/config/log4j2.xml

# Location of the services definition (OIDC clients definitions)
cas.serviceRegistry.config.location: file:/etc/cas/services

# OIDC provider configuration
cas.authn.oidc.issuer=https://cas.example.com:8443/cas/oidc
cas.authn.oidc.jwksFile=file:/etc/cas/keystore.jwks

## We define a custom scope *profile_full* to retrieve our user attributes
cas.authn.oidc.scopes=openid,profile,given_name,email,profile_full

## We map the custom scope *profile_full* to our claims and attributes
cas.authn.oidc.userDefinedScopes.profile_full=employeeNumber,givenName,lastName,mail,displayName
cas.authn.oidc.claimsMap.employeeNumber=employeeNumber
cas.authn.oidc.claimsMap.givenName=cn
cas.authn.oidc.claimsMap.lastName=sn
cas.authn.oidc.claimsMap.mail=mail
cas.authn.oidc.claimsMap.displayName=displayName

cas.authn.oidc.claims=sub,name,preferred_username,family_name, \
given_name,middle_name,given_name,profile, \
picture,nickname,website,zoneinfo,locale,updated_at,birthdate, \
email,email_verified,phone_number,phone_number_verified,address, \
givenName,lastName,mail,displayName,employeeNumber

# LDAP configuration
cas.authn.ldap[0].principalAttributeList=uid,mail,cn,sn,employeeNumber,displayName
cas.authn.ldap[0].allowMissingPrincipalAttributeValue=true
cas.authn.ldap[0].type=AUTHENTICATED
cas.authn.ldap[0].ldapUrl=ldap://localhost:10389
cas.authn.ldap[0].useSsl=false
cas.authn.ldap[0].baseDn=ou=people,dc=example,dc=edu
## We authenticate using the uid attribute
cas.authn.ldap[0].userFilter=uid={user} 
cas.authn.ldap[0].bindDn=cn=Directory Manager
cas.authn.ldap[0].bindCredential=password
```

{% include note.html content="Here the LDAP bind password is specified in plain text. Check [this blog post](https://apereo.github.io/2017/03/24/cas51-ldapauthnjasypt-tutorial) if you want to specify specify an encrypted value." %}

### Build CAS webapp

```bash
cd cas-overlay-template
mvn clean package
```

Then you just need to deploy the target/cas.war file in your favourite web server.

### Keystore configuration

To be able to sign the OIDC token, we need to configure a JSON Web Key Set (JWKS).

```bash
git clone https://github.com/mitreid-connect/json-web-key-generator.git
cd json-web-key-generator
mvn package
cd target
java -jar json-web-key-generator-0.4-SNAPSHOT-jar-with-dependencies.jar -t RSA -s 2048 -i 1 -u sig -S -o /etc/cas/keystore.jwks
```

### OIDC client registration

OpenID Connect clients can be registered using the JSON registry service. You need to create a file in /etc/cas/services folder with the following naming convention :

```javascript
fileName = serviceName + "-" + serviceNumericId + ".json"
```

I created a file named *demoOIDC-207929965088748.json* with the following content :

```json
{
  "@class": "org.apereo.cas.services.OidcRegisteredService",
  "clientId": "demoOIDC",
  "clientSecret": "password",
  "serviceId": "^https://app.example.com/redirect",
  "signIdToken": true,
  "implicit": true,
  "bypassApprovalPrompt": false,
  "name": "Demo app",
  "id": 207929965088748,
  "evaluationOrder": 100,
  "encryptIdToken": false,
  "scopes": [ "java.util.HashSet",
    [ "openid", "profile", "profile_full" ]
  ]
}
```

I will use the custom scope *profile_full* to retrieve all user attributes.

## Let's test !

We will perform a implicit authorization flow.

```javascript
https://cas.example.com:8443/cas/oidc/authorize?response_type=id_token%20token&client_id=demoOIDC&scope=openid%20profile%20profile_full&redirect_uri=https%3A%2F%2Fapp.example.com%2Fredirect&state=3km36n5yp2l9h26&nonce=po7s2tr6wnc8xs2
```

We are correclty redirected to the CAS login page. I use my jdoe user to authenticate.

<table>
<tr>
<td><img src="images/login.png"/> </td>
<td><img src="images/consent.png"/></td>
</tr>
</table>

{% include note.html content="I modified the WEB-INF/messages.properties file to add the description to the custom scope profile_full" %}

After consenting to the requested scopes, we are redirected to the application with the tokens in the url query parameters.

We can succesfully retrieve all claims inside the id_token :

```json
{
  "jti": "af322620-accc-4a2c-b71e-c1d30f6a87c7",
  "iss": "https://cas.example.com:8443/cas/oidc",
  "aud": "demoOIDC",
  "exp": 1500232568,
  "iat": 1500225368,
  "nbf": 1500225068,
  "sub": "jdoe",
  "amr": [
    "LdapAuthenticationHandler"
  ],
  "state": "3km36n5yp2l9h26",
  "nonce": "po7s2tr6wnc8xs2",
  "at_hash": "VCOZ4tr3FZ07BYjIqD6qAQ==",
  "displayName": "John Doe",
  "employeeNumber": "123456789",
  "givenName": "John",
  "lastName": "Doe",
  "mail": "john.doe@mail.com",
  "preferred_username": "jdoe"
}
```
