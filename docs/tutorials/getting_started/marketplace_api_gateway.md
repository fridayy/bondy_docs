---
outline: [2,3]
draft: true
related:
    - text: Marketplace
      type: Tutorial
      link: /tutorials/getting_started/marketplace
      description: A tutorial that demonstrates a simple marketplace with Python microservices and a VueJS Web App.
    - text: HTTP API Gateway
      type: Reference
      link: /reference/api_gateway/specification#api-specification-object
      description: Bondy HTTP API Gateway acts as a reverse proxy by accepting incoming REST API actions and translating them into WAMP actions over a Realm's procedures and topics.
    - text: HTTP API Gateway
      type: HTTP API Reference
      link: /reference/http_api/api_gateway
      description: Load and manage API Gateway specifications.
---

# Marketplace HTTP API Gateway
A tutorial that demonstrates how to add an HTTP/REST API to an existing project using the HTTP API Gateway.

## Goal
In this tutorial we are going to pick up where we left off with the [Marketplace tutorial](/tutorials/getting_started/marketplace).

The idea is to now expose the WAMP API we have developed to HTTP. The goal is to demonstrate how one can use Bondy's capabilities to integrate HTTP clients into the application network.

The steps in the following sections will demonstrate how to create an HTTP [API Specification Object](/reference/api_gateway/specification#api-specification-object) from scratch, loading it in Bondy and demonstrating how we can call the resulting HTTP/REST API using an HTTP client.

## Background

The architecture remains the same as the one discussed in [Marketplace tutorial](/tutorials/getting_started/marketplace#demo-architecture) with the exception that we will now have the HTTP API Gateway and an HTTP client which are shown in the updated view below.

<ZoomImg src="/assets/tutorials/marketplace_api_gateway/marketplace.png"/>

The [Marketplace](/tutorials/getting_started/marketplace) offers the following WAMP API procedures registered on the `com.example.demo` realm:

- `com.market.bidder.add` - Add a user as bidder. Takes a single positional argument, the bidder name
- `com.market.bidder.gone` - Remove a user as bidder. Takes a single positional argument, the bidder name
- `com.market.get` - List all the items for sale
- `com.market.item.bid` - Place a bid on a listed item. Takes three positional arguments:
    - item name
    - item new price
    - bidder name
- `com.market.item.get` - Return an item's details. Takes single positional argument, the item name
- `com.market.item.sell` - List a new item for sale. Takes three positional arguments:
    - item name
    - item price
    - item deadline (in minutes)

:::::: warning Note
Since we wrote the Marketplace tutorial we have updated it so that the `com.example.demo` realm is already configured to support for OAuth2 authentication:
* `oauth2` and `password` authmethods enabled
* users, groups and grants to be able to perform the OAuth2 flow.

So make sure you **update you local copy** of the Marketplace Demo repository to the latest.

In addition, the realm had the following users defined:

- `postman`: Postman app user for testing purposes that belongs to the `api_clients` group
- `victor`: An end user for testing purposes that belongs to the `resource_owners` group

::: details Check the changes we made to the realm
You can see them in the lines highlighted below.

```json 5-6
{
    "authmethods": [
        "cryptosign",
        "anonymous",
        "oauth2",
        "password"
    ],
    "description": "The market realm",
    "is_prototype": false,
    "is_sso_realm": false,
    "password_opts": {
        "params": {
            "iterations": 10000,
            "kdf": "pbkdf2"
        },
        "protocol": "cra"
    },
    "public_keys": [
        {
            "crv": "P-256",
            "kid": "11116844",
            "kty": "EC",
            "x": "-wc2zdqixwOixJ45Bi_bmr5WknFyUibrQMWkLXbL3Kg",
            "y": "YBWzCcHCGMDAmUfx84J3O53L7QcZl_Be4zEMsjQmq8g"
        },
        {
            "crv": "P-256",
            "kid": "119815566",
            "kty": "EC",
            "x": "ce3F2nAqePIAgJawlupqUE3gjVmTSYLrAc76wyToBPc",
            "y": "a0WppYd2m_7UpXbb1SfW0O3i7USuxbUokl48Uk6GARQ"
        },
        {
            "crv": "P-256",
            "kid": "129179117",
            "kty": "EC",
            "x": "0XSBmbwcOTKxR14Ic8Nr9LMAjgOz_3FYfg7wJCaEdl4",
            "y": "TU9ji-7AelCmwrOu0fNfVN0G8o6LeNf-rNqjDspq2lM"
        }
    ],
    "security_status": "enabled",
    "uri": "com.market.demo"
}
```
:::

::::::


## 1. Run the Marketplace Demo

Follow the instructions in the [Marketplace tutorial](/tutorials/getting_started/marketplace#_2-build-and-run-the-demo).

## 2. Defining an API Gateway Spec
In this part we try to show how we can create and configure the api spec to be able to expose some endpoints to call registered wamp procedures:

### 2.1. Define an API Object

As a first step we need to declare and [API Object](/reference/api_gateway/specification#api-object). In this case we are configuring it with the **id** `com.market.demo` on the **realm** `com.market.demo` and with **oauth2** as security enabled and also some other defaults and configuration.

Example below:
```json
{
    "id":"com.market.demo",
    "name":"Marketplace Demo API",
    "host":"_",
    "realm_uri":"com.market.demo",
    "meta":{
    },
    "variables":{
        "oauth2":{
            "type":"oauth2",
            "flow":"resource_owner_password_credentials",
            "token_path":"/oauth/token",
            "revoke_token_path":"/oauth/revoke",
            "schemes":"{{variables.schemes}}"
        },
        "query_spec":"{{request.query_params |> with([_q,_p,_limit,_sort,_page,_include,_from,_to])}}",
        "query_match":"{{request.query_params |> without([_q,_p,_limit,_sort,_page,_include,_from,_to])}}",
        "schemes":[
            "http"
        ],
        "forward_spec":{
            "action":{
                "type":"forward",
                "host":"{{variables.host}}"
            },
            "response":{
                "on_error":{
                    "body":"{{variables.wamp_error_body}}"
                },
                "on_result":{
                    "uri":"{{action.result.uri}}",
                    "body":"{{action.result.body}}"
                }
            }
        },
        "cors_headers":{
            "access-control-allow-origin":"*",
            "access-control-allow-credentials":"true",
            "access-control-allow-methods":"GET,HEAD,OPTIONS,PUT,PATCH,POST,DELETE",
            "access-control-allow-headers":"origin,x-requested-with,content-type,accept,authorization,accept-language",
            "access-control-max-age":"86400"
        },
        "wamp_error_body":"{{action.error.kwargs |> put(code, {{action.error.error_uri}})}}"
    },
    "defaults":{
        "retries":0,
        "timeout":15000,
        "connect_timeout":5000,
        "schemes":"{{variables.schemes}}",
        "security":"{{variables.oauth2}}",
        "headers":"{{variables.cors_headers}}"
    },
    "status_codes": {
        "com.example.error.not_found": 404,
        "com.example.error.unknown_error": 500,
        "com.example.error.internal_error": 500
    },
    ...
}
```

2. Then we need to define the [Version Object](/reference/api_gateway/specification#version-object) and its path. In this case we are defining the **version** `1.0.0` and it is part of the default path (optional).

Example below:

```json
{
    ..
    "versions": {
        "1.0.0":{
            "base_path":"/[v1.0]",
            "variables":{
                "host":"http://localhost:8080"
            },
            "defaults":{
                "timeout":20000
            },
            "languages":[
                "en"
            ],
            "paths":{
                 ...
            }
        }
    }
}
```

3. Then we need to define the paths using the [Path Object](/reference/api_gateway/specification#path-object) with the proper HTTP method, action type, WAMP procedures, arguments and responses. Example below for the `/market` GET endpoint calling to the proper `com.market.get` WAMP procedure without any args nor kwargs. Example below:

::: info Note
In the configured path you can notice, for example, the property `is_collection: true` due to the result of this endpoint is a list of items
:::

```json
{
     ...
     "versions": {
         "1.0.0":{
             ...
             "paths":{
                "/market":{
                    "is_collection":true,
                    "options":{
                        "action":{

                        },
                        "response":{
                            "on_error":{
                                "body":""
                            },
                            "on_result":{
                                "body":""
                            }
                        }
                    },
                    "get":{
                        "action":{
                            "type":"wamp_call",
                            "procedure":"com.market.get",
                            "options":{
                            },
                            "args":[
                            ],
                            "kwargs":{
                            }
                        },
                        "response":{
                            "on_error":{
                                "status_code":"{{status_codes |> get({{action.error.error_uri}}, 500) |> integer}}",
                                "body":"{{variables.wamp_error_body}}"
                            },
                            "on_result":{
                                "body":"{{action.result.args |> head}}"
                            }
                        }
                    }
                }
             }
         }
     }
}
```

## 3. Loading the API Gateway Spec

Load the defined api spec using the [Bondy Administrative API](/reference/http_api/index)

```bash
curl -X "POST" "http://localhost:18081/services/load_api_spec" \
-H 'Content-Type: application/json; charset=utf-8' \
-H 'Accept: application/json; charset=utf-8' \
--data-binary "@api_gateway_config.json"
```

If the loading and applying was OK, th curl hasn’t result; detailed response with code and message in failure cases.

From the Bondy side, you can see in the logs the following lines:

```erlang
2022-12-06 10:48:11 bondy | when=2022-12-06T13:48:11.600576+00:00 level=notice pid=<0.1855.0> at=bondy_http_gateway:rebuild_dispatch_tables/0:223 description="Rebuilding HTTP Gateway dispatch tables" node=bondy@127.0.0.1 router_vsn=1.0.0-beta.68
2022-12-06 10:48:11 bondy | when=2022-12-06T13:48:11.604499+00:00 level=info pid=<0.1855.0> at=bondy_http_gateway:load_dispatch_tables/0:830 description="Loading and parsing API Gateway specification from store" node=bondy@127.0.0.1 router_vsn=1.0.0-beta.68 timestamp=-576460723125 name="Marketplace Demo API" id=com.market.demo
```


### Execution and test

1. Getting an oauth token ([JWT](https://jwt.io/)) using password method
    - Request

        ```bash
        curl --location --request POST 'http://localhost:18080/oauth/token' \
        --header 'Content-Type: application/x-www-form-urlencoded' \
        --header 'Authorization: Basic cG9zdG1hbjpQb3N0bWFuMTIzNDU2IQ==' \
        --data-urlencode 'username=victor' \
        --data-urlencode 'password=Victor123456!' \
        --data-urlencode 'grant_type=password'

        # where 'cG9zdG1hbjpQb3N0bWFuMTIzNDU2IQ==' is the base64 enconding of postman:Postman123456!
        ```

    - Response example

        ```json
        {
            "access_token": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJjb20ubWFya2V0LmRlbW8iLCJleHAiOjkwMCwiZ3JvdXBzIjpbInJlc291cmNlX293bmVycyJdLCJpYXQiOjE2NzAzMzQ2ODAsImlkIjoxMjEwNjU1ODA2NTU3MzIsImlzcyI6InBvc3RtYW4iLCJraWQiOiI4MzMwOTAxMSIsIm1ldGEiOnsiZGVzY3JpcHRpb24iOiJWaWN0b3IgZW5kIHVzZXIgZm9yIHRlc3RpbmcgcHVycG9zZXMifSwic3ViIjoidmljdG9yIn0.fQ0Ctl9RV4TYzfcusHy1f6aDuqIVuMkffx08vJ9dFq-x8at3fdZR3alCrF1I2lYT5vJFA7YJqjHPb-rbDB2Y1A",
            "expires_in": 900,
            "refresh_token": "tlI4SzKqVsjcUsXYCXXmA2JkjAKZ1T7QSX5GB5Or",
            "scope": "resource_owners",
            "token_type": "bearer"
        }
        ```

2. Invoking an endpoint using Bearer Token method with the retrieved token above
    - Example `/market/item` endpoint trying to sell a new item
        - Request executing a POST method

            ```bash
            curl --location --request POST 'http://localhost:18080/market/item' \
            --data-raw '{
                "name": "first item",
                "price": 1,
                "deadline": 1
            }'
            ```

        - Response where the item was published successfully

            ```json
            true
            ```

    - Example of `/market` endpoint trying to retrieve the market items
        - Request executing a GET method

            ```bash
            curl --location --request GET 'http://localhost:18080/market' \
            --header 'Authorization: Bearer eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJjb20ubWFya2V0LmRlbW8iLCJleHAiOjkwMCwiZ3JvdXBzIjpbInJlc291cmNlX293bmVycyJdLCJpYXQiOjE2NzAzMzQ2ODAsImlkIjoxMjEwNjU1ODA2NTU3MzIsImlzcyI6InBvc3RtYW4iLCJraWQiOiI4MzMwOTAxMSIsIm1ldGEiOnsiZGVzY3JpcHRpb24iOiJWaWN0b3IgZW5kIHVzZXIgZm9yIHRlc3RpbmcgcHVycG9zZXMifSwic3ViIjoidmljdG9yIn0.fQ0Ctl9RV4TYzfcusHy1f6aDuqIVuMkffx08vJ9dFq-x8at3fdZR3alCrF1I2lYT5vJFA7YJqjHPb-rbDB2Y1A'
            ```

        - Response

            ```json
            [
                {
                    "deadline": "2022-12-06T14:06:10.306988+00:00",
                    "name": "first item",
                    "price": 591,
                    "winner": "Mary"
                }
            ]
            ```

    - Example of `/market/bidder` endpoint trying to add a new bidder
        - Request executing a POST method

            ```bash
            curl --location --request POST 'http://localhost:18080/market/bidder' \
            --header 'Authorization: Bearer eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJjb20ubWFya2V0LmRlbW8iLCJleHAiOjkwMCwiZ3JvdXBzIjpbInJlc291cmNlX293bmVycyJdLCJpYXQiOjE2NzAzMzU3OTMsImlkIjo2NDI1NTI3ODE5NTYxMTQsImlzcyI6InBvc3RtYW4iLCJraWQiOiIyMDkxODg3MSIsIm1ldGEiOnsiZGVzY3JpcHRpb24iOiJWaWN0b3IgZW5kIHVzZXIgZm9yIHRlc3RpbmcgcHVycG9zZXMifSwic3ViIjoidmljdG9yIn0.6Cm25Hs9lCMP4OwXPeRei6kYc1E2YuGjj4yaQ1bP9D5_LI0VeUny0c2AMyYOO76MyCvXGFRJjM3uVhREvA1obw' \
            --header 'Content-Type: application/json' \
            --data-raw '{
                "name": "ale"
            }'
            ```

            From the Market side, you can see in the logs the following lines:

            ```python
            2022-12-06 11:14:27 market  | New bidder: ale
            ```

    - Example of `/market/bidder/:name` trying to delete a registered bidder
        - Request executing a DELETE method

            ```bash
            curl --location --request DELETE 'http://localhost:18080/market/bidder/ale' \
            --header 'Authorization: Bearer eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJjb20ubWFya2V0LmRlbW8iLCJleHAiOjkwMCwiZ3JvdXBzIjpbInJlc291cmNlX293bmVycyJdLCJpYXQiOjE2NzAzMzU3OTMsImlkIjo2NDI1NTI3ODE5NTYxMTQsImlzcyI6InBvc3RtYW4iLCJraWQiOiIyMDkxODg3MSIsIm1ldGEiOnsiZGVzY3JpcHRpb24iOiJWaWN0b3IgZW5kIHVzZXIgZm9yIHRlc3RpbmcgcHVycG9zZXMifSwic3ViIjoidmljdG9yIn0.6Cm25Hs9lCMP4OwXPeRei6kYc1E2YuGjj4yaQ1bP9D5_LI0VeUny0c2AMyYOO76MyCvXGFRJjM3uVhREvA1obw' \
            --header 'Content-Type: application/json' \
            --data-raw '{
                "name": "ale"
            }'
            ```

            From the Market side, you can see in the logs the following lines:

            ```python
            2022-12-06 11:17:31 market  | ale left, cancel thier bids.
            ```

3. Invoking an endpoint with invalid credentials
    1. Using an old token with an inexistent user we have the following response:

        ```json
        {
            "code": "wamp.error.no_such_principal",
            "description": "",
            "message": "The request failed because the authid provided does not exist."
        }
        ```

2. Using an expired token we have the following response:

```json
{
    "code": "invalid_grant",
    "description": "The provided authorization grant (e.g., authorization code, resource owner credentials) or refresh token is invalid, expired, revoked, does not match the redirection URI used in the authorization request, or was issued to another client. The client MAY request a new access token and retry the protected resource request.",
    "message": "The access or refresh token provided is expired, revoked, malformed, or invalid."
}
```


## Conclusion

It is very easy to configure and expose an HTTP/REST API on top of an existing WAMP API. We can do it by using a declarative specification language based on JSON and without any coding.

As a reference, you can see the resulting HTTP API Specification file at [api_gateway_config.json](https://github.com/bondy-io/bondy-demo-marketplace/blob/main/resources/api_gateway_config.json).

Finally, you can check the complete [Marketplace Postman Collection](/assets/tutorials/Bondy_Marketplace_demo.postman_collection.json).

<ZoomImg alt="Postman Collection overview" src="/assets/tutorials/Postman_Collection.png"/>