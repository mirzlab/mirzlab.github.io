---
layout: post
title: Password blacklist with Forgerock IDM 5
summary: How to prevent users from choosing a widely known password using a password blacklist
---

NIST recently released new password guidelines, one of those being:

*\"When processing requests to establish and change memorized secrets, verifiers SHALL compare the prospective secrets against a list that contains values known to be commonly-used, expected, or compromised.\"*

In this blog post we will implement a blacklist mechanism using IDM 5 to prevent users from choosing a password that is easily vulnerable to a dictionary attack.

-----

# Implementation

## Define a password blacklist

Let\'s create our list of password that need to be banned:


```sh
$ cd openidm/bin/default/scripts
$ touch policy-passwordBlacklist.js
```

Edit the newly created script with the following code:

```javascript
(function () {
  exports.passwordList = [
    "123456",
    "123456789",
    "qwerty",
    "12345678",
    "111111",
    "1234567890",
    "1234567",
    "password",
    "123123",
    "987654321",
    "qwertyuiop",
    "mynoob",
    "123321",
    "666666",
    "18atcskd2w",
    "7777777",
    "1q2w3e4r",
    "654321",
    "555555",
    "3rjs1la7qe",
    "google",
    "1q2w3e4r5t",
    "123qwe",
    "zxcvbnm",
    "1q2w3e"
  ]
}());
```

{% include note.html content="This list is based on [Keeper Most Common Passwords of 2016](https://keepersecurity.com/public/Most-Common-Passwords-of-2016-Keeper-Security-Study.pdf)." %}

{% include note.html content="If your password blacklist is very long, you might want to use a map instead of an array for better performance. You will need to change the checking function too (see below)." %}

## Password policy configuration

Open the default IDM 5 policy script file:

```sh
$ vi openidm/bin/defaults/script/policy.js
```

Inside the *policies* array variable, add a new policy reference to make it available for password verification:

```json
{   
  "policyId" : "not-in-most-common-password-list",
  "policyExec" : "notInMostCommonPasswordList",
  "clientValidation": true,
  "validateOnlyIfPresent": true,
  "policyRequirements" : ["NOT_IN_MOST_COMMON_PASSWORD_LIST"]
}
```

Then create the actual function that will perform the check:


<pre style="font-size: small">
policyFunctions.notInMostCommonPasswordList = function(fullObject, value, params, prop) {
  var passwordListPolicy = require("policy-passwordBlacklist");
  var isRequired = _.find(this.failedPolicyRequirements, function (fpr) {
    return fpr.policyRequirement === "REQUIRED";
  }),
  isNonEmptyString = (typeof(value) === "string" && value.length),
  valueIsNotInList = isNonEmptyString ? 
    (passwordListPolicy.passwordList.indexOf(value) < 0) : false;

  if ((isRequired || isNonEmptyString) && !valueIsNotInList) {
    return [ { "policyRequirement" : "NOT_IN_MOST_COMMON_PASSWORD_LIST" } ];
  }

  return [];
};
</pre>

Next step is to tell IDM to use this policy for the password attribute of the user object.

```sh
$ vi openidm/conf/managed.json 
```

Edit the *user* object by applying the the new policy to the password attribute:

```
"password" : {
    "title" : "Password",
    "type" : "string",
    "viewable" : false,
    "searchable" : false,
    "minLength" : 8,
    "userEditable" : true,
    "encryption" : {
        "key" : "openidm-sym-default"
    },
    "scope" : "private",
    "isProtected": true,
    "policies" : [
        {
            "policyId" : "not-in-most-common-password-list"
        },
        {
            "policyId" : "cannot-contain-others",
            "params" : {
                "disallowedFields" : [
                    "userName",
                    "givenName",
                    "sn"
                ]
            }
        }
    ]
}
```

Last thing, we want to display a nice error message in case a user tries to choose one of the banned password.

```sh
$ vi openidm/ui/selfservice/default/locales/en/translation.json
```

Edit the *common.form.validation* object by adding our policy reference:

```
"common": {
  "form": {
...
    "validation" : {
      "NOT_IN_MOST_COMMON_PASSWORD_LIST": "This password is too weak. Please choose another password.",
      ...
  }
}

```

## Let\'s test 

We can test the policy by performing a password reset flow with a choosen user.

After receiving an email with a link inside, IDM asks me to choose another password.

If I choose a banned password, the policy is triggered and prevents the password change:

<img src="/images/password-blacklist.png" style="width: 100%; height: auto"/>
