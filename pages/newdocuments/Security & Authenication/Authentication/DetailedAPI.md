---
pagename: Detailed API
redirect_from:
  - guides-authentication-detailedapi.html
sitesection: Documents
categoryname: "Security & Authentication"
documentname: Authentication

permalink: authentication-detailed-api.html
order: 5
indicator: both
---

### Web Interaction Embedded Window API

In order to enable targeting for messaging engagements, configure the CustomerId Engagement Attribute in order to identify the visitor utilizing LivePerson’s backend authentication services. The CustomerId variable is part of CustomerInfo Engagement Attribute set. It is not used for visitor authentication, but as a trigger to LivePerson monitoring services to start targeting and sending relevant engagements and/or notifications to the visitor.

By attributing the conversation to the CustomerID, new incoming messages will be delivered and displayed as a window in a minimized state, with new message notifications.

In this use case, it is the Customer’s Web App responsibility to set the customerId and generate a valid token. The LivePerson Web SDK calls a JavaScript method located on the page, and provides it with a callback method to execute when it has a token as a response to LivePerson Web Tag, and is able to continue the flow.

The callback method accepts two parameters:

*	token - a string token. Alternatively an object can be provided containing 2 properties: "ssoKey" - a string token, and "redirect_uri" - a string URI (relevant for embedded code flow only).

*	error - any value except null or undefined to describe the error that has occurred

The Customer web page method name can be either the default LivePerson method name (lpGetAuthenticationToken), or any specified name which can be accessed by traversing the global scope on the page.

![Web Interaction Embedded Window API](img/webinteraction.png)

**Code Example**

```javascript
    log("set customerId");
    lpTag.sdes.push({
        "type": "ctmrinfo",
        "info": {
            "customerId": customerId
        }
    });

    var lpMethods = {
        lpGetAuthenticationToken: function(callback) {
            log("LP asked for id_token");
            // Do your magic...
            // On Success
            callback(id_token);
            // On Failure
            callback("", "error reason");
        },
        // Or, if you want to provide a redirect_uri as well (instead of the default "https://liveperson.net")
        lpGetAuthenticationTokenWithRedirectURI: function(callback) {
            log("LP asked for id_token");
            // Do your magic...
            // On Success
            callback({ssoKey: id_token, redirect_uri: uri});
            // On Failure
            callback({}, "error reason");
        }
    };
```

### OpenID Connect provider (OP) development details

**API Request**

* The interface is implemented as REST/JSON.  In keeping with the REST specification, the verb is POST, since accessing this API changes the state on the server (is not idempotent).

*	LivePerson will POST the following data using the "application/x-www-form-urlencoded" format:

  code=b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c
  grant_type=authorization_code
  redirect_uri=https://liveperson.net

*	The token (JWT) should contain three base64url encoded segments separated by period ('.') characters.

*	The following HTTP headers are required: header name: “Authorization" The value contains the standard basic authorization header [RFC2617], based on client_id:client_secret provided by the team to identify.

_Example: Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW_

```
POST /token HTTP/1.1   

Host: server.example.com   
Content-Type: application/x-www-form-urlencoded   
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW    
grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https%3A%2F%2Fliveperson.net%2Fcb
```

**API Response**

If the transaction is successful, then the response will be an HTTP 200, and the following payload:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store   
Pragma: no-cache    
{    
	"access_token"	: "NotApplicabale",
  	"token_type"		: "Bearer",
	"id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MjU1NTUxMjEyIiwiY29uc3VtZXJfdHlwZSI6ImNvcHBlciIsImlzcyI6Imh0dHBzOi8vd3d3LnQtbW9iaWxlLmNvbSIsImV4cCI6MTQ0NjExNTM1MjAwMCwiaWF0IjoxNDQ2MTExNzUyMDAwfQ.mMERThLDcoW51434sJyOtRUOZZVOVmB_evxTwduJ1Ht1Q78ZZ6ZLqH3tN3idXI7-Qn7WOwej2OI-8vsAENB_7gxxpZFUlQ8dCZFM1o7ZJd5gsXvjHbHgIlnRn1zxonZ5L8pIO8TByTNOgwDp847JyGStyzEZTYKkyOwxB5p96Z8"
```

The id_token is a standard JSON web token (see http://jwt.io) [RFC 7519], with the following data:

```json
{
"sub": "4255551212",
"iss" : "https://www.YourBrand.com",
"exp" : 1446115352,
"iat" : 1446111752,
"extended claims set  - ref: OpenIdConnect token structure"
}
```
The following table describes the response fields:

|    Field    |    Description                                                                                                                             |
|-------------|--------------------------------------------------------------------------------------------------------------------------------------------|
|    sub      |    The consumer ID of the authenticated subscriber.                                                                                      |
|    iss      |    The name of the Authorization   Service as configured in LivePerson.                                                                    |
|    exp      |    When LivePerson should   ask for a new token (validating that the user is still logged in). Seconds   from 1970, UTC. see [RFC3339]     |
|    iat      |    When this JWT was   issued. Seconds from 1970, UTC. see [RFC3339]                                                                       |

The id_token will be signed using RS256 ALG (see http://jwt.io) [RFC 7519], and the public key for verifying it will be supplied to LivePerson during the configuration stage.

### Error Response

If an error is encountered during processing, an error message will be returned:

```
HTTP/1.1 400 Bad Request
Content-Type: application/json   
Cache-Control: no-store   
Pragma: no-cache    
{    "error": "invalid_code"   }
```

One of the following errors will be returned:

|    Name                   | Description                                                               |  
|---------------------------|---------------------------------------------------------------------------|
|    json_parse_error       |    There was a problem   with the format of the JSON document provided.   |
|    invalid_code           |    The code provided   is not valid.                                      |
|    unauthorized_client    |    The token was valid,   but this subscriber is not authorized to use the messaging feature.    Note that this should not be encountered in the production environment,   as the check is performed on the client before the LivePerson SDK is invoked. |

### OpenID Token structure

LivePerson supports the following claims set, which will be displayed to the agent after successful authentication.

**Mandatory Claim Set**:

|    Name    |    Description                                                            |    Type      |    LP_Name    |
|------------|---------------------------------------------------------------------------|--------------|---------------|
|    iss     |    The name of the Authorization Service as configured in LivePerson.   |    string    |    -          |
|    iat     |    When this JWT was issued by your Service. Seconds from 1970, UTC. See [RFC3339]. |    number     |    -          |
|    exp     |    When LivePerson should   ask for a new token (validating that the user is still logged in). Seconds   from 1970, UTC. See [RFC3339].  |    number    |    -     |

**Standard Claim Set**:


|    Name    |    Description | type   |    Appearance in LiveEngage   |    Mapping in SDEs    |
|------------|-------------------|-------------------------------|-----------------------|
|sub         |Subject - Identifier for the end-user at the Issuer.|string| Consumer info (including Customer ID)| ConsumerInfo.customerID|
|given_name      |Given name(s) or first name(s) of the end-user. Note that some people may have multiple given names; all can be present, with the names being separated by space characters.|string| Personal info (including Name) | Personalinfo.firstname|
|family_name        |Surname(s) or last name(s) of the end-user. Note that some people may have multiple family names or no family name; all can be present, with the names being separated by space characters.|string| Personal info (including Name)| Personalinfo.lastname|
|email        |End-User's preferred e-mail address. Its value MUST conform to the RFC 5322 [RFC5322] addr-spec syntax. The RP MUST NOT rely upon this value being unique, as discussed in Section 5.7.|string| Personal info (including Email address)| Personalinfo.personalcontact.email|
|gender        |End-user's gender. Values defined by this specification are female and male. Other values MAY be used when neither of the defined values are applicable|string| Personal info (including Gender)| Personalinfo.gender|
|preferred_username       |Shorthand name by which the end-user wishes to be referred by at the RP such as janedoe or j.doe. This value MAY be any valid JSON string including special characters such as @, /, or whitespace. The RP MUST NOT rely upon this value being unique.|string| Consumer info (including User Name)| CustomerInfo.username|
|phone_number      |End-user's preferred telephone number. E.164 [E.164] is RECOMMENDED as the format of this Claim, for example, +1 (425) 555-1212 or +56 (2) 687 2400. If the phone number contains an extension, it is recommended that the extension be represented using the RFC 3966 [RFC3966] extension syntax, for example, +1 (604) 555-1234;ext=5678.|string| Personal info (including Phone number)| PersonalInfo.contacts[].phone|

**Example for Mandatory+Standard Claims JWT**:

```json
{
"sub": "4255551212",
"iss" : "https://www.YourBrand.com",
"exp" : 1446115352,
"iat" : 1446111752,
"preferred_username" : "JohnDoe",
"phone_number" : "+1-10-344-3765333"
}
```

**Custom Claim**:

Custom Claims will be added in LP SDE form from the SDEs list, see the [Engagement Attributes Overview document](engagement-attributes-overview.html) for more info on the possible Engagement Attributes. (**Please note that some Engagement Attributes are NOT supported in an authenticated flow. Please see the document for details on which attributes are supported and which aren't**).

Example for Mandatory+Standard+Custom Claims JWT:

```json
{  
   "sub":"4255551212",
   "iss":"https://www.YourBrand.com",
   "exp":1446115352,
   "iat":1446111752,
   "preferred_username":"JohnDoe",
   "phone_number":"+1-10-344-3765333",
   "lp_sdes":[  
      {  
         "type":"ctmrinfo",
         "info":{  
            "cstatus":"cancelled",
            "ctype":"vip",
            "customerId":"138766AC",
            "balance":-400.99,
            "socialId":"11256324780",
            "imei":"3543546543545688",
            "userName":"user000",
            "companySize":500,
            "accountName":"bank corp",
            "role":"broker",
            "lastPaymentDate":{  
               "day":15,
               "month":10,
               "year":2014
            },
            "registrationDate":{  
               "day":23,
               "month":5,
               "year":2013
            }
         }
      }
   ]
}
```

### OpenID JWT Encryption

*	Encryption ensures that only the addressee will be able to read it (for example the consumer itself won't be able to read the content.)

*	In order to encrypt the JWT payload, use a public key supplied by LivePerson.

*	The key is base64 decoded with X509 key spec.

*	The encryption should use RSA algorithm.

*	JWE header should include header name “alg" with the value: RSA_OAEP_256.

### OpenID JWT Signing

*	Signature proves the authenticity of the issuer.

*	The JWT should be signed with your RSA private key.

*	The public key should be added to LivePerson OAuth configuration in the “JWT Public Key" field, and should be base64 encoded with X509 key spec.

### Extract JWT Public and Private Keys

Conduct the following three steps to receive private and public keys:

1.	Generate RSA key:

```sh
openssl genrsa -out private_idp.pem 2048
```

2.	Extract private key:

```sh
openssl pkcs8 -topk8 -inform pem -in private_idp.pem -outform pem -nocrypt -out private_key_idp.pem
	(private key can be found in the file: private_key_idp.pem)
```

3.	Extract public key:

```sh
openssl rsa -in private_idp.pem -outform PEM -pubout -out public_key_idp.pem
(public  key can be found in the file  : public_key_idp.pem)
```

Configure the JWT public key on LiveEngage UI:

1.	Remove the header and tail ( "-----BEGIN PUBLIC KEY-----" & "-----END PUBLIC KEY-----" )

Example:

```sh
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA/lnYC8lHMdEJz74m2WHr
K8ndDE3LDto1xZ8HlyOwLWZWkvy71t43zzyMD1cpDfsymv6fSZSfP1NkwF6k9eTW
RMYaK0TVivmAXdL4GoO/87XpPiEPn/lxICWL4VTJN/ju1by+h4qVlcpevJic4ZEK
5PXF1vhHqlawPcm/IsWRcYAkFb/Mk84d+i7+OQ7NK0ouqb1T57ijUUGTQEqpzLLo
TWZ7wjlWnyteuHx2OtRwskfH/3U7A7GP9STFZDC0yyDh7bj9DSuI4ScQWhmXMRy9
OT/vxvJBf+pNrlNqto+dxNwJtSPKUQ0cnugdiI2mzw40HRtyQxzQONyuttdEMzMX
6wIDAQAB
-----END PUBLIC KEY-----
```
2.	Wrap text of code to be structured as continuous line (remove any new line char).

Example:

```
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA/lnYC8lHMdEJz74m2WHrK8ndDE3LDto1xZ8HlyOwLWZWkvy71t43zzyMD1cpDfsymv6fSZSfP1NkwF6k9eTWRMYaK0TVivmAXdL4GoO/87XpPiEPn/lxICWL4VTJN/ju1by+h4qVlcpevJic4ZEK5PXF1vhHqlawPcm/IsWRcYAkFb/Mk84d+i7+OQ7NK0ouqb1T57ijUUGTQEqpzLLoTWZ7wjlWnyteuHx2OtRwskfH/3U7A7GP9STFZDC0yyDh7bj9DSuI4ScQWhmXMRy9OT/vxvJBf+pNrlNqto+dxNwJtSPKUQ0cnugdiI2mzw40HRtyQxzQONyuttdEMzMX6wIDAQAB
```

3. Paste the public key into the LiveEngage authentication server set up:

![Authentication Setup](img/authenticationserversetup.png)