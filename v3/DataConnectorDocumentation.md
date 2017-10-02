# Data Connector (Waiter service) - Documentation

## Table of Content
  * [Terms of use - TAUS membership](#terms-of-use---taus-membership)
  * [Server Info](#server-info)
  * [Authentication](#authentication)
  * [Core Concepts](#core-concepts)
    + [Action](#action)
    + [Scope](#scope)
    + [Filters](#filters)
    + [Groups](#groups)
    + [Other Actions](#other-actions)
    + [Project Related Requests](#project-related-requests)
  * [Basic Attributes](#basic-attributes)
    + [Benchmarking](#benchmarking)

This document is intended for anyone who is interested in using the DQF Data Connector (DC)  in order to receive metrics and statistics related to DQF projects tracked through various integrations via the DQF API (https://github.com/TAUSBV/dqf-api/tree/master/v3). DC is a public web API and its responses can be used to create custom dashboards with project reports as well as user, organization and industry statistics. TAUS Quality Dashboard (http://qd.taus.net/) is such a client application, consuming the DC as a service.

This documentation will explain the core concepts of the DC and give some examples on how to use the service.

For detailed API specifications, please visit:
https://data-connector.stag.taus.net

For a step-by-step tutorial, please visit:
https://github.com/TAUSBV/dqf-data-connector/blob/master/v3/DataConnectorTutorial.md

For any questions related to the usage of the DC API, please contact dqfapi@taus.net

## Terms of use - TAUS membership
DC API is available to TAUS Business level subscribers or higher.

## Server Info

During your testing process, you should use the DQF Staging Server:

API:<br/>
https://dqf-api.stag.taus.net/ <br/>
This is the general DQF API on the DQF Staging Server, and is typically used for development of CAT-tool plugins to supply the DQF database with data. This API is not a part of DC, and is explained in more detail here: https://github.com/TAUSBV/dqf-api/blob/dev/v3/README.md

Quality Dashboard:<br/>
https://qd.stag.taus.net/ <br/>
This is the TAUS client of DC on the staging server; the Quality Dashboard is the most popular way of interacting with the DQF data.

Data Connector:<br/>
https://data-connector.stag.taus.net <br/>
This is the root URL for the requests on the Staging server. Visit this URL for information on each of the endpoints in the Data Connector API in the Swagger interface.

The staging server is dedicated to integrators only. All new features and/or fixes are deployed here before going to the DQF production server. In order to use the staging server, you first need to request a test account. Please write to dqfapi@taus.net

Once your tests are completed, you should contact the DQF team in order to enable access of your application to the production server. After this, you should switch your base URLs to our official ones:

Site: <br/>
https://taus.net

API: <br/>
https://dqf-api.taus.net

Quality Dashboard: <br/>
https://qd.taus.net

Data Connector: <br/>
https://data-connector.taus.net

## Authentication
You will need the following credentials to identify your requests:

* API key
* Encryption key
* initVector

Every request must contain the header parameter `apiKey`, in the format of a Universally Unique Identifier (UUID), which will be provided by TAUS. The apiKey is application specific and used to identify the client that sends the requests. TAUS provides apiKeys that are specific for each of its services. The apiKey for the Data Connector cannot be used for requests at DQF-API, and vice versa. Every integrator will have one apiKey.

For secured endpoints, there should also be a header parameter `sessionId`. In order to obtain a sessionId, you must call the POST /v3/login endpoint.

The body parameters (`email`, `password`) represent the user credentials - encrypted and Base64 encoded.

The encryption algorithm used is AES/CBC/PKCS5PADDING. The encryption key will also be provided by TAUS and will be valid on both servers (staging and production).

Below you see a simple Java code snippet for the encryption part using the javax.crypto lib:

```
public static String encrypt(String value, String key) throws Exception {
    IvParameterSpec iv = new IvParameterSpec(initVector.getBytes("UTF-8"));
    SecretKeySpec skeySpec = new SecretKeySpec(key.getBytes("UTF-8"), "AES");

    Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5PADDING");
    cipher.init(Cipher.ENCRYPT_MODE, skeySpec, iv);

    byte[] encrypted = cipher.doFinal(value.getBytes("UTF-8"));

    return Base64.encodeBase64String(encrypted);
}
```

The initVector will also be provided by TAUS. The initVector will remain the same for the production environment. Should you decide to use your own initialization vector, it should be 16 Bytes and you must provide TAUS with it.

For testing/debugging purposes, we have enabled an encrypt endpoint which is accessible through POST /v3/encrypt on https://dqf-api.stag.taus.net . No authentication is required. The body parameters are:

* `email`: The user's email as plain text
* `password`: The user's password as plain text
* `key`: The encryption key

With a successful request, you should get back your encrypted and Base64 encoded credentials.

IMPORTANT: The encrypt endpoint is not available in production and should not be used as part of your final implementation.

## Response Content Type
All the responses are in JSON format. You should explicitly handle the status of the response in order to retrieve additional information. For example, in .NET you can catch a WebException for a BadRequest and parse the content to see what went wrong:

```
{
   "code": 6,
   "message": {
      "password": [
         "This field is required"
      ]
   }
}
```

The code is DQF specific and should be used to report the nature of the problem to the DQF team.

NOTE: For debugging and troubleshooting purposes, it is critical that you can provide logs of the calls to the DQF server. The DQF Team strongly recommends to include UI elements containing the API responses in case of errors. Errors in the API communication need to be reported to the DQF Team. Also, inform the user of the application about the error by an error message, ideally also when operations are concluded successfully.

## Core Concepts
DC responses are dynamic and can provide customized results based on how the URL is constructed.
There are four core concepts in the DC which are used to construct the URL:

* [Action](#action)
* [Scope](#scope)
* [Filters](#filters)
* [Groups](#groups)

See this example of a request to the DC:
```
{host}/v3/productivity/industry?targetLanguage=fr-FR,de-DE&segmentOrigin=TM&groupBy=contentType
```

`{host}` stands for the root urls https://data-connector.staging.taus.net or https://data-connector.taus.net .

Here, productivity is the action, industry is the scope, we applied target language and segment origin filters and we grouped by content type.

Let’s see what the request in the example above returns as a response. The request to the DC service can be rephrased like this: “Give me productivity related metrics for the whole industry, only for French and German as target languages, considering only segments with Translation Memory as origin and group my results by content type.” The response to the request could be:

```
[
    {
        "group": {
            "contentType": "Informative content"
        },
        "totalTime": 18452065
    },
    {
        "group": {
            "contentType": "Knowledge Base/Glossary"
        },
        "totalTime": 83919
    },
    {
        "group": {
            "contentType": "Marketing Material"
        },
        "totalTime": 3528
    },
    {
        "group": {
            "contentType": "Online Help"
        },
        "totalTime": 548390
    },
    {
        "group": {
            "contentType": "Other"
        },
        "totalTime": 404094
    },
    {
        "group": {
            "contentType": "Technical Specifications"
        },
        "totalTime": 1144
    },
    {
        "group": {
            "contentType": "Training Material"
        },
        "totalTime": 655336
    },
    {
        "group": {
            "contentType": "User Interface Text"
        },
        "totalTime": 735647
    },
    {
        "group": {
            "contentType": "User Manual"
        },
        "totalTime": 17350873
    }
]
```

### Action

In DC, response models are tied to Actions. In the example above the response model is simple and looks like this:

**ProductivityResponse {**<br/>
&nbsp;&nbsp;&nbsp;**group** (Group, *optional*),<br/>
&nbsp;&nbsp;&nbsp;**totalTime** (integer, *optional*)<br/>
**}**<br/>
**Group {}**

And an example JSON value:

```
{
  "group": {},
  "totalTime": 0
}
```

All response models contain the group attribute which is populated based on the usage of the Groups (explained later).

Currently available actions that can be parameterized with scopes/filters/groups are:

* annotationErrors
* productivity
* projectCount
* projectList
* reviewCorrection
* sourceCount
* translationCount

In other words, an action represents the higher level of information we are interested in.

### Scope
Scope represents the type of aggregation that will happen for the specified action. There are four types of scope:

 1. Project
 2. User
 3. Organization
 4. Industry

Most of DC requests support all 4 scopes and this is important for benchmarking purposes. Project scope returns results aggregated on project level. User scope returns results for everything related to the authenticated user. If a user belongs to an organization and has the proper rights, he/she should be able to use the organization scope which aggregates for all the organizational users. Finally there is the industry scope which requires no authentication and aggregates everything that is present in the DQF database.

### Filters
Here is where the actual customization begins. Filters (and Groups) are optional parameters in the URL and can be used to dynamically change the response’s contents. Every action has an allowed list of filters (see specifications) and groups. Filters limit the results based on their content. There can be multiple filters in one request and every filter can contain an array of values. In the example above, the filters were:

`targetLanguage=fr-FR,de-DE&segmentOrigin=TM`

With this we limited the aggregation only for projects with French and German as target languages and for segments that originated from a Translation Memory.

### Groups
Similar to **Filters**, **Groups** can be used to customize the response but in a different way: Based on the grouping(s) that were applied, the response will be broken down to an array of elements, one for every distinct grouping combination. In the aforementioned example our group was `groupBy=contentType`.

The response contained a productivity element for every content type that matched the applied filters. Groups can contain an array of values, meaning that we can apply multiple groupings in a request. For example we can request a group of

`groupBy=contentType,segmentOrigin `

and therefore results will be broken down for every possible combination of content type and segment origin.

### Other Actions
There are some requests that are standalone and cannot be customized with filters and groups. These usually contain a large amount of information regarding general statistics for projects, users and organizations.

### Project Related Requests
For the requests using the project scope, the URL also contains the **project’s Id** (ex. `{host}/v3/reviewCorrection/project/5755`). There is also an additional header that must be used: `projectKey`. Both of these values can be retrieved by using

```
/v3/projects/user
```

```
/v3/projects/organization
```

to get a user’s/organization’s list of projects. It is recommended not to expose the project’s key to the client side of your application.

## Basic Attributes
In order to get complete lists for the filter values, you can use the DQF API’s (not the DC!) respective endpoints.
These endpoints are used to retrieve the basic/static attributes from the DQF API. No authentication is required for these. The basic attributes can be grouped according to their function. More details will be provided in the related sections:

 - GET /v3/catTool<br/>Return the Computer-assisted translation tools
 - GET /v3/contentType<br/>Get the content types for the project set up
 - GET /v3/errorCategory<br/>Get list of top category errors in error hierarchy
 - GET /v3/industry<br/>Get the industries list for the project set up
 - GET /v3/mtEngine<br/>Return the MT Engines
 - GET /v3/process<br/>Get the Productivity processes for the project set up
 - GET /v3/qualitylevel<br/>Get the translation quality levels for the project set up
 - GET /v3/segmentOrigin<br/>Get list of segment origin values
 - GET /v3/severity<br/>Get list of error severities

The following call is no longer necessary after DQF has become compliant with the BCP47 standard:

- GET /v3/language
Get a language list from DQF

### Benchmarking
The true potential of the DC is the benchmarking possibility for different scopes. For example you can benchmark a project’s productivity with the user’s, organization or industry respective metrics by using the same customization for filters and groups. Taking another example we can create a chart with many series/datasets, one for each scope by just changing the scope in the URL. We can request the values for the time in each scope, filter and grouping:

```
{host}/v3/productivity/user?industry=Construction%2FReal+Estate%2CAerospace%2FAviation&groupBy=targetLanguage,segmentOrigin
```

```
{host}/v3/productivity/organization?industry=Construction%2FReal+Estate%2CAerospace%2FAviation&groupBy=targetLanguage,segmentOrigin
```

```
{host}/v3/productivity/industry?industry=Construction%2FReal+Estate%2CAerospace%2FAviation&groupBy=targetLanguage,segmentOrigin
```

Please note that most of the DC’s endpoints return raw metrics and not calculated values. For example, in order to produce a chart on productivity (productivity = (sourceWordCount * 3,600,000) / totalTime as words per hour) we need to make the appropriate requests to get the totalTime from the productivity action and also use the sourceCount action to get source segments/words/character counts:

```
{host}/v3/sourceCount/user?industry=Construction%2FReal+Estate%2CAerospace%2FAviation&groupBy=targetLanguage,segmentOrigin
```

```
{host}/v3/sourceCount/organization?industry=Construction%2FReal+Estate%2CAerospace%2FAviation&groupBy=targetLanguage,segmentOrigin
```

```
{host}/v3/sourceCount/industry?industry=Construction%2FReal+Estate%2CAerospace%2FAviation&groupBy=targetLanguage,segmentOrigin
```

This will produce the chart below in the TAUS Quality Dashboard (test data):
![Screenshot productivity chart with benchmarking](https://github.com/TAUSBV/dqf-waiter/blob/dev/CaptureBMproductivity.PNG?raw=true)

