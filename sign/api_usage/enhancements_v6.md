# Adobe Sign v6 REST API Enhancements

## On this page 
- [Overview](#overview)
- [Enhancements](#enhancements)
    - [Pagination support ](#paginationsupport)
    - [Constant secure IDs](#constantsecureids)
    - [ETag support](h#etagsupport)
    - [GET, PUT, POST consistency](#getputpostconsistency)
    - [Performance improvements](#performanceimprovements)
- [Features](#features)
    - [Agreement sharing](#agreementsharing)
    - [Authoring APIs](#authoringapis)
    - [Document visibility](#documentvisibility)
    - [Draft](#draft)
    - [Notes management](#notesmanagement)
    - [Reminders](#reminders)
    - [Resource view](#resourceview)
    - [Resource visibility](#resourcevisibility)
    - [Stamp support](#stampsupport)
    - [Suppress email](#suppressemail)
    - [Webhooks](#webhooks)
- [API change log](api_change_log.md)

## Overview

This page is a comprehensive guide for Adobe developers who are looking to plug Adobe Sign Version 6 REST APIs in their solutions. We list out all the enhancements and new features that will be part of the v6 Adobe Sign APIs. To facilitate developers who are already using older versions of our APIs in migrating to Version 6, we have tabulated the v6 APIs against their closest counterparts in previous versions in our [API Change Log](api_change_log.md). In the change log, we have also documented enhancements, features, and any changes from the prior version. In addition to the information compiled here, you can also refer to our  [Swagger documentation page](https://secure.echosign.com/public/docs/restapi/v6) to get a quick reference for the Adobe Sign APIs. This page lists all our APIs in a easily discoverable format and lets you try them out without having to write any code!

## Enhancements

### Pagination Support

Existing Adobe Sign APIs return the entire list of resources (agreements, widgets, and library documents) that a user has in a GET call. For any user with even decent transactions, this list soon becomes huge. The v6 APIs have introduced pagination support to all these resources with a client-configurable page size. This will be especially useful to our mobile clients and integrations who are constrained by their consumption capacity. This sample shows pagination in a request and response:

**Sample Request**

```GET https://api.na1.echosign.com:443/api/rest/v6/agreements?pageSize=50```

**Sample Response**

```json
{
    "userAgreementList": [{
      "displayDate": "2017-04-17T06:07:19-07:00",
      "displayUserSetInfos": [
        {
          "displayUserSetMemberInfos": [
            {
              "company": "TestAccNaman",
              "email": "es.auto+a2@hotmail.com",
              "fullName": "a2 Admin"
            }
          ]
        }
      ],
      "esign": true,
      "agreementId": "3AAABLblqZhDqIcUs4nFgivebIUdzuZyBrjO_VP_hHDhkrGhXxKuQ5Hi7C07vRbNzxP9TdTdRHzHdQLDsPJrjfXEuKe7jjEAl",
      "latestVersionId": "3AAABLblqZhACieamyoCl7qNWZTaU3WaoY3a9BL7-09sosH88HyRFfGmYc91jpQk-LXLVGlgEudioxgPlCprAScifamX16-QD",
      "name": "SampleAgreement",
      "status": "SIGNED"
    },
    {...},
    .
    .
    .],
    "page": {
        "nextCursor": "qJXXj2UAUX1X9rTSqoUOlkhsdo*"
    }
}
```

The subsequent GET /resources calls would just need to add  **nextCursor**  as  **query param**  in the URL for fetching the next set of resources.

**Sample Request**

```GET https://api.na1.echosign.com:443/api/rest/v6/agreements?pageSize=50&cursor=qJXXj2UAUX1X9rTSqoUOUOlkhsdo*```

### Constant secure IDs

A major issue that our partners and integrators have reported is that the identifiers (IDs) of our resources can change. Partner integrations tend to store these resource IDs and do a match later, which can break with ID changes. This can be due to the encryption algorithm changing or key rotation. Also, at present our IDs are tied to the API caller (which means the same resource can have different IDs) and can get fairly long due to encapsulating resource information. The v6 APIs address this by ensuring that IDs stay constant through time and across all API callers for a given resource. This will enable clients to build their own metadata associated with an Adobe Sign resource ID.

We do guarantee the forward capability for this change: in other words, **older identifiers will continue working in v6 Sign APIs.** However, new identifiers generated through v6 APIs **will only be compatible with Adobe Sign v6 or any higher version.** Therefore, we do not recommend partial upgrade to v6 APIs as it might break things.

### ETag support

Polling on a resource is a common operation; for instance, in the case of agreements, where a client application is attempting to check the latest status. In fact, the count of corresponding REST and SOAP APIs for this polling supersedes the entire remaining set of calls combined. The v6 REST APIs significantly optimize this by adding support for an ETag, which will only fetch the full body response if the resource has been modified. This saves the client from loading the same response as well as its unnecessary parsing.

In addition, this also helps to resolve conflicts in the case of concurrent update operations by only allowing the updates with the ETag of the latest version of the resource in their request header (_If-match_).  This example explains the functioning of ETags in detail:

**Sample GET Operation (ETag in Request Header)**

```http
URI : GET https://api.na1.echosign.com:443/api/rest/v6/agreements/CBJCHBCAABAAQonMXhG-V6w-rheRViZNFGxmCgEEf3k0 

Headers : Authroization: Bearer <access-token>
          Accept: */*
          Accept-Encoding: gzip, deflate, br
          Accept-Language: en-US,en
          If-None-Match: CBJCHBCAABAA-mdO9PI7WFmHNkXFUIEYIOYGrnM3vVK_
```

The above request will result in a **304(Resource Not Modified)** HTTP response if the ETag provided in the If-None-Match header represents the latest version of the resource. Otherwise, the response body will include the queried resource in the usual format, along with an ETag representing the fetched version of the resource in the response header.

The _response header_ below represents the second scenario, in which the resource has been modified prior to the request (notice the ETag as one of the headers).

**Response Header**
```http
Date: Mon, 12 Feb 2018 09:45:24 GMT
Server: Apache
ETag: D27e5290dc3a748068e42a59f4dfc6f6b1d5eaba1
Content-Length: 661
Strict-Transport-Security: max-age=31536000;
X-XSS-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Robots-Tag: noneP3P: CP="IDC DSP COR ADM DEVi TAIi PSA PSD IVAi IVDi CONi HIS OUR IND CNT"
Keep-Alive: timeout=15, max=200
Connection: Keep-Alive
Content-Type: application/json;charset=UTF-8
```

Update or delete operations will have a similar workflow. The clients are required to provide the ETag of the resource version that they want to update in the If-Match request header. The update will only be successful if the ETag in the request header represents the latest version of the resource on the server. Otherwise, this will result in a **412 (Precondition Failed)** response. In this example, we are trying to update the status of the agreement fetched above:

**Sample PUT Operation (ETag In Request Header)**

```http
URI : PUT  https://api.na1.echosign.com:443/api/rest/v6/agreements/CBJCHBCAABAAQonMXhG-V6w-rheRViZNFGxmCgEEf3k0/status

Headers : Authorization: Bearer <access-token>
          Accept: */*
          Accept-Encoding: gzip, deflate, br
          Accept-Language: en-US,en
          If-Match: CBJCHBCAABAA-mdO9PI7WFmHNkXFUIEYIOYGrnM3vVK_ 
```

The response below indicates that we are trying to update an older version (_observe the ETag in the request_) of this resource. Along with this response body, the response header contains the HTTP status code 412(Precondition Failed).

**Sample PUT Operation (ETag In Request Header)**

```json
{
    "code": "RESOURCE_MODIFIED",
    "message": "Resource is already modified with newer version"
}
```

The ETag value required to be passed in any PUT or DELETE API can be obtained from a corresponding GET operation on the same entity. The table below mentions these modification (PUT or DELETE) APIs along with the corresponding GET APIs that provides the ETag value for these modification requests.

| Update/Deletion API | Corresponding GET endpoint |
| --- | --- |
| PUT /agreements/{agreementId} | GET /agreements/{agreementId} |
| PUT /agreements/{agreementId}/formFields | GET /agreements/{agreementId}/formFields |
| PUT /agreements/{agreementId}/formFields/mergeInfo | GET /agreements/{agreementId}/formFields/mergeInfo |
| PUT /agreements/{agreementId}/members/participantSets/{participantSetId} | GET /agreements/{agreementId}/members/participantSets/{participantSetId} |
| PUT /agreements/{agreementId}/members/participantSets/{participantSetId}/participants/{participantId}/acknowledgement | GET /agreements/{agreementId}/members/participantSets/{participantSetId} |
| PUT /agreements/{agreementId}/members/participantSets/{participantSetId}/participants/{participantId}/formFieldValues | GET /agreements/{agreementId}/members/participantSets/{participantSetId}/participants/{participantId}/formFieldValues |
| PUT /agreements/{agreementId}/members/participantSets/{participantSetId}/participants/{participantId}/reject | GET /agreements/{agreementId}/members/participantSets/{participantSetId} |
| PUT /agreements/{agreementId}/members/participantSets/{participantSetId}/participants/{participantId}/status | GET /agreements/{agreementId}/members/participantSets/{participantSetId} |
| PUT /agreements/{agreementId}/state | GET /agreements/{agreementId} |
| DELETE /agreements/{agreementId}/documents | GET /agreements/{agreementId}/documents |
| PUT /libraryDocuments/{libraryDocumentId} | GET /libraryDocuments/{libraryDocumentId} |
| PUT /libraryDocuments/{libraryDocumentId}/state | GET /libraryDocuments/{libraryDocumentId} |
| PUT /widgets/{widgetId} | GET /widgets/{widgetId} |
| PUT /widgets/{widgetId}/state | GET /widgets/{widgetId} |
| PUT /megaSigns/{megaSignId}/state | GET /megaSigns/{megaSignId} |
| DELETE /users/{userId}/signatures/{signatureId} | GET /users/{userId}/signatures/{signatureId} |
| PUT /webhooks/{webhookId} | GET /webhooks/{webhookId} |
| PUT /webhooks/{webhookId}/state | GET /webhooks/{webhookId} |
| DELETE /webhooks/{webhookId} | GET /webhooks/{webhookId} |

### GET, PUT, POST consistency

Current REST APIs, due to their tight coupling with SOAP, have a different interface for POST (resource creation), GET (fetching), and PUT (update). This can complicate clients who have to manage different models for each of these operations. The Version 6 REST APIs solve this by explicitly focusing on GET, PUT and POST consistency in our interfaces. For  more elaboration on this please refer to the  [agreement model](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/getAgreementInfo) in GET, PUT and POST operations.

### Performance improvements

The internal re-architecture of the backend implementation and decoupling of SOAP and REST has allowed us to take a fresh look at the implementation and get rid of redundant checks and calls to the database. This, in general, has resulted in the improvement of the client experience. Also, we have migrated to the asynchronous creation of resources, which has significantly improved our response time. The client experience of using the APIs has significantly improved due to these performance enhancmenents.

However, due to the asynchronous nature of the creation APIs, the clients must **poll** on the status of the created resource before fetching _certain_ sub-resources(for example, documents in the case of agreements) or performing any modifications on the resource. For example, in case of agreement creation, the initial status is `DOCUMENTS_NOT_YET_PROCESSED` which is updated to the intended status such as `OUT_FOR_SIGNATURE` once all the background tasks are successfully completed.

## Features

### Agreement sharing

This feature enables users associated with an agreement to share the agreement at any point of time through Adobe Sign APIs. This feature brings the agreement sharing capability in Adobe Sign web app and Adobe Sign APIs up to par. The  [POST /agreements/{agreementId}/members/share](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/createShareOnAgreement) API exposes the agreement sharing feature.

### Authoring APIs

The authoring APIs are a set of APIs that allow a user to _author_ the documents of an agreement before sending them out. The authoring operation here refers to creating, editing or placing form fields along with their configurations (assignee, conditions, data type, and more) in the agreement documents. The v6 APIs have these capabilities and a client can now exploit these APIs to create their own agreement authoring experience. The table below lists the APIs in this set along with the functionality that they provide.

| **Authoring API** | **Functionality** |
| --- | --- |
| [POST /agreements/{agreementId}/formFields](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/postFormFields) | Adds forms to an agreement from the given template. The response would contain the information of all the newly added form fields. |
| [GET /agreements/{agreementId}/formFields](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/getFormFields) | Retrieves all the form fields present in an agreement. |
| [PUT /agreements/{agreementId}/formFields](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/updateFormFields) | Updates and configures(say location, default value, background, etc.) the present form fields in the agreement documents. |
| [DELETE /agreements/{agreementId}/formFields](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/deleteFormFields) | Removes a _list_ of form fields identified by their name from an agreement. |

### Document visibility

The new document visibility feature allows senders to control the exposure of agreement documents for specific participants through APIs. This empowers clients to hide from participants those parts of agreements that are irrelevant to them. The agreement creation request below hides the second document of the agreement from the first participant.

**Document Visibility Example**

```json
{
    "fileInfos": [{
            "transientDocumentId": "<first-transient-document>"
        },
        {
            "transientDocumentId": "<second-transient-document>"
        }
    ],
    "name": "Custom Agreement",
    "participantSetsInfo": [{
            "memberInfos": [{
                "email": "firstSigner@adobe.com"
            }],
            "name": "First Signer",
            "visiblePages": [
                "0", "1"
            ],
            "order": 1,
            "role": "SIGNER"
        },
        {
            "memberInfos": [{
                "email": "secondSigner@adobe.com"
            }],
            "name": "Second Signer",
            "visiblePages": [
                "0", "1", "2"
            ],
            "role": "SIGNER",
            "order": "2"
        }
    ],
    "signatureType": "ESIGN",
    "state": "IN_PROCESS"
}
```

### Draft

One of the problem that clients have faced while creating resources (say, agreements) was that the creation process needed to be single-shot. The incremental creation of any resource is really helpful, especially when it constitutes one of many complex components. To achieve this capability, we have introduced the concept of draft for all resources. Draft is a temporary or primitive stage of the final intended resource that can be updated in steps to create the final resource.

This example illustrates a stepwise creation of an agreement:

**Step 1: POST /agreements to create an initial draft**

```json
{
    "fileInfos": [{
        "transientDocumentId": "<a-valid-transient-resource-id>"
    }],
    "name": "Draft",
    "signatureType": "ESIGN",
    "state": "DRAFT"
}
```

The step above creates a draft. Notice that we have not assigned any participant to this agreement yet.

**Step 2: PUT /agreements/agreementId to complete this draft**

```json
{
    "fileInfos": [{
        "transientDocumentId": "<a-valid-transient-resource-id>"
    }],
    "name": "Agreement",
    "participantSetsInfo": [{
        "memberInfos": [{
            "email": "signer@adobe.com"
        }],
        "role": "SIGNER",
        "order": "1"
    }],
    "signatureType": "ESIGN",
    "state": "DRAFT"
}
```

Notice the addition of a participant and an update in the `name` field. This step can be iterated any number of times until we have all the data needed to create the agreement.

The next step finalizes the draft into an agreement.

**Step 3: PUT /agreements/&lt;agreementId&gt;/state to complete this draft/**

```json{
  "state": "IN_PROCESS"
}
```

### Notes management

The v6 Adobe Sign APIs has endpoints to manage notes in an agreement. Clients can add notes to an agreement and retrieve them using these API's. The table below lists all these APIs and their operation.



| **Notes API** | **Functionality** |
| --- | --- |
| [GET /agreements/{agreementId}/me/note](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/getAgreementNoteForApiUser) | Retrieves the latest note on an agreement for the user. |
| [GET /libraryDocuments/{libraryDocumentId}/me/note](https://secure.echosign.com/public/docs/restapi/v6#!/libraryDocuments/getLibraryDocumentNoteForApiUser) | Retrieves the latest note on a library template for the user. |


### Reminders

The reminder APIs in v6 enable clients to create reminders for _any_ participant at any time before their action on the agreement. The capability to list all reminders on an agreement is also availaible in v6. These capabilities will significantly improve clients&rsquo; experience of handling reminders for agreements. The table below lists all the endpoints in this set:

| **Authoring API** | **Functionality** |
| --- | --- |
| [POST /agreements/{agreementId}/reminders](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/createReminderOnParticipant) | Sets reminders for a list of participant. |
| [GET /agreements/{agreementId}/reminders](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/getAgreementReminders) | Retrieves all the reminders set in an agreement. |

### Resource view

There are a number of _views_ associated with a resource. For example, an agreement may have an authoring view, agreement documents view, signing page view, or a manage page view with the agreement selected. The availaibility of all these views depends on both the state of the resource and also the relationship of the user with the resource. To access and parameterize these resource views, the v6 Adobe Sign API includes this endpoint to list all such views in their desired configuration:  
[POST /resource/{resourceId}/views](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/getAgreementView)

**Sample request/response:**

_Request - POST /agreements/{agreementId}/views_

```java
{
    "names": "DOCUMENT",
    "commonConfiguration": {
        "autoLoginUser": false,
        "noChrome": true,
        "locale": "en"
    }
}
```

_Response - POST /agreements/{agreementId}/views_

```json
[  
    {
        "name": "DOCUMENT",
        "url": "https://secure.echosign.com/account/agreements?aid=CBJCHBCAABAA0RVdUCYoR5kU9vh4-b4qHhYW_1r10hKw&pid=CBJCHBCAABAAH-F0jK3mHa53G7gr0SiftgdqE-jjwNVq&client_id=pUQL757H2R2&noChrome=true",
        "embeddedCode": "<script type='text/javascript' language='JavaScript' src='https://secure.echosign.com/embed/account/agreements?aid=CBJCHBCAABAA0RVdUCYoR5kU9vh4-b4qHhYW_1r10hKw&pid=CBJCHBCAABAAH-F0jK3mHa53G7gr0SiftgdqE-jjwNVq&client_id=pUQL757H2R2&noChrome=true'></script>"
    }
]
```

### Resource visibility

The agreement visibility feature enables a client to control which resources are included in the response body of the enumeration/reource listing APIs. This helps users to hide all resources from their view that they don&rsquo;t want to focus on. The  [PUT /resource/{resourceId}/me/visibility](https://secure.echosign.com/public/docs/restapi/v6#!/agreements/updateAgreementVisibility) API exposes this functionality, wherein a resource can be an agreement, widget, template or megasign in Adobe Sign.

### Suppress email

The suppress email feature, in a broader sense, enables us to specify which emails participants receive while sending out the agreement. This feature is exposed through our agreement creation API, `POST /agreements`. Here is a sample request that allows only agreement initiation emails to be sent to the participants:

**POST /agreements with email configuration**

```json
{
    "fileInfos": [{
        "transientDocumentId": "<a-valid-transient-resource-id>"
    }],
 
    "name": "Sample Agreement with email config",
 
    "participantSetsInfo": [{
        "memberInfos": [{
            "email": "signer@adobe.com"
        }],
        "role": "SIGNER",
        "order": "1"
    }],
       
    "emailOption": {
        "sendTarget": {
            "initEmails": "ALL",
            "inFlightEmails": "NONE",
            "completionEmails": "NONE"
        }
    },
       
    "signatureType": "ESIGN",
    "status": "IN_PROCESS"
}
```

### Webhooks

Callbacks in Adobe Sign are now handled through **webhooks**. A _webhook_ is essentially a web service designed to listen for and respond to POST requests. When you create a webhook and register it with Adobe Sign, Adobe Sign's Webhook Subscription Service will notify your webhook of any relevant event by sending a POST request via HTTPS containing a JSON object with the details of the event. Your webhook then passes those details on to your application for handling. The service operates on a push model: your app doesn't have to poll for events at all&mdash;those events are automatically sent to your webhook as they happen, with virtually no delay, so your app is instantaneously, automatically updated with any changes. 

See [Webhooks in Adobe Sign v6](../webhooks.md) to learn how webhooks work in Adobe Sign and how to set one up for your application.
