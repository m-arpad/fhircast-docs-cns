<img style="float: left;padding-right: 5px;" src="/img/hl7-logo-header.png" width=90px" />
# FHIRcast
## !! Modified !!
Please note that this is a modified version of the [original FHIRCast specification](http://fhircast.org/specification/STU1/). It contains extensions proposed by the Cerner-Nuance-Siemens Collaboration in order to

1. Support Near-Real-Time Exchange of Clinical Case information to be used in various integration scenarios between different systems.
2. Improve the robustness, security and deployability of the solution in various networks environments including those protected by firewalls and proxies.
3. Decrease latency of event notifications

The major additions/changes to the standard are:

* Added 'Sharing of Structured Information in a Case'
* Added 'update-case' event
* Replaced WebSub-based service-to-client communication with WebSockets

## Overview
The FHIRcast specification describes the APIs and interactions to synchronize healthcare applications in real time to show the same clinical content to a user. All data exchanged through the HTTP APIs MUST be sent and received as [JSON](https://tools.ietf.org/html/rfc8259) structures, and MUST be transmitted over channels secured using the Hypertext Transfer Protocol (HTTP) over Transport Layer Security (TLS), also known as HTTPS and defined in [RFC2818](https://tools.ietf.org/html/rfc2818). FHIRcast utilizes [WebSockets](https://github.com/HL7/fhircast-docs/wiki/Websocket-proposal) for bi-directional communication, specifically all calls from service to client. FHIRcast also builds on the [HL7 SMART on FHIR launch protocol](http://www.hl7.org/fhir/smart-app-launch).

An app subscribes to specific workflow events for a given session, the subscription is verified and the app is notified when those workflow events occur; for example, by the clinician opening a patient's chart. The subscring app may query a session's current context and initiate context changes by accessing APIs exposed by the Hub. The app deletes its subscription when it no longer wants to receive notifications. In all cases the app authenticates to the Hub with an OAuth 2.0 bearer token. 

## Session Discovery
Before establishing a subscription, an app must know the `hub.topic` which is an unique url identifying the session, and the `cast-hub` which is the base url of the Hub. The app discovers these two urls as part of a SMART on FHIR launch. 

The app MUST either be launched from the driving application following the [SMART on FHIR EHR launch](http://www.hl7.org/fhir/smart-app-launch#ehr-launch-sequence) flow or the app may initiate the launch following the [SMART on FHIR standalone launch](http://www.hl7.org/fhir/smart-app-launch/#standalone-launch-sequence). In either case, the app MUST request and be granted the `fhircast` OAuth2.0 scope. Accompanying this scope grant, the authorization server MUST supply the `cast-hub` and `hub.topic` SMART launch parameters alongside the access token. These parameters identify the Hub's base url, and a unique, opaque identifier of the current user's session, respectivly. Per SMART, when scopes of `openid` and `fhirUser` are granted, the app will additionally receive the current user's identity in an `id_token`.

### SMART Launch Example
Note that the SMART launch parameters include the Hub's base url and and the session identifier in the `cast-hub` and `hub.topic` fields.

```
{
  "access_token": "i8hweunweunweofiwweoijewiwe",
  "token_type": "bearer",
  "expires_in": 3600,
  "patient":  "123",
  "encounter": "456",
  "imagingstudy": "789",
  "cast-hub" : "https://hub.example.com",
  "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
}
```
Although FHIRcast works best with the SMART on FHIR launch and authorization process, implementation-specific launch, authentication, and authorization  protocols may be possible. See [other launch scenarios](/launch-scenarios) for guidance.

## Subscribing and Unsubscribing

Subscribing consists of two phases:

* Subscriber requests a subscription at the `cast-hub` url and as a response receives a websocket address to connect to.
* Subscriber connects to the websocket address and starts to listen for messages being sent to it through the websocket channel.

Unsubscribing works in the same way, except with a single parameter changed to indicate the desire to unsubscribe.

### Subscription Request
To create a subscription, the subscribing app SHALL perform an HTTP POST ([RFC7231](https://www.w3.org/TR/websub/#bib-RFC7231)) to the Hub's base url (as specified in `cast-hub`) with the parameters in the table below.

This request SHALL have a `Content-Type` header of `application/x-www-form-urlencoded` and SHALL use the following parameters in its body, formatted accordingly:

Field | Optionality | Type | Description
---------- | ----- | -------- | --------------
`hub.mode` | Required | *string* | The literal string "subscribe" or "unsubscribe", depending on the goal of the request.
`hub.topic` | Required | *string* | The uri of the user's session that the subscriber wishes to subscribe to or unsubscribe from. 
`hub.events` | Required | *string* | Comma-separated list of event types from the Event Catalog for which the Subscriber wants notifications.
`hub.channel.type` | Required | *string* | The literal string "websocket"
`hub.lease_seconds` | Optional | *number* | Number of seconds for which the subscriber would like to have the subscription active, given as a positive decimal integer. Hubs MAY choose to respect this value or not, depending on their own policies, and MAY set a default value if the subscriber omits the parameter. 

If OAuth2 authentication is used, this POST request SHALL contain the Bearer access token in the HTTP Authorization header.

Hubs SHALL allow subscribers to re-request subscriptions that are already activated. Each subsequent and verified request to a Hub to subscribe or unsubscribe SHALL override the previous subscription state for a specific topic / callback URL combination.

#### Subscription Request Example
In this example, the app asks to be notified of the `open-patient-chart` and `close-patient-chart` events.
```
POST https://hub.example.com
Host: hub.example.com
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/x-www-form-urlencoded

hub.mode=subscribe&hub.topic=https%3A%2F%2Fhub.example.com%2F7jaa86kgdudewiaq0wtu&&hub.events=patient-open-chart,patient-close-chart&hub.channel.type=websocket
```

### Subscription Response
If the Hub URL supports FHIRcast and is able to handle the subscription or unsubscription request, the Hub MUST respond to a subscription request with an HTTP 202 "Accepted" response to indicate that the request was received. The response MUST contain the WebSocket URI that the client will need to connect to in order to start receiving notifications.

If a Hub finds any errors in the subscription request, an appropriate HTTP error response code (4xx or 5xx) MUST be returned. In the event of an error, the Hub SHOULD return a description of the error in the response body as plain text, used to assist the client developer in understanding the error. This is not meant to be shown to the end user. Hubs MAY decide to reject some clients or topic URIs based on their own policies.

#### Subscription Response Example
```
HTTP/1.1 202 Accepted

wss://hub.example.com/ee30d3b9-1558-464f-a299-cbad6f8135de
```

### Subscription Denial

If (and when) the subscription is denied, the Hub SHALL inform the subscriber by sending an HTTP GET request to the subscriber's callback URL as given in the subscription request. This request has the following query string arguments appended, to which the subscriber SHALL respond with an HTTP success (2xx) code.

Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.mode` | Required | *string* | The literal string "denied".
`hub.topic` | Required | *string* | The topic uri given in the corresponding subscription request.
`hub.events` | Required | *string* | A comma-separated list of events from the Event Catalog corresponding to the events string given in the corresponding subscription request. 
`hub.reason` | Optional | *string* | The Hub may include a reason for which the subscription has been denied. The subscription MAY be denied by the Hub at any point (even if it was previously accepted). The Subscriber SHOULD then consider that the subscription is not possible anymore.

#### Subscription Denial Example
```
GET https://app.example.com/session/callback/v7tfwuk17a?hub.mode=denied&hub.topic=https%3A%2F%2Fhub.example.com%2F7jaa86kgdudewiaq0wtu&hub.events=open-patient-chart,close-patient-chart&hub.reason=session+unexpectedly+stopped HTTP 1.1
Host: subscriber
```

### Unsubscribe

Once a subscribing app no longer wants to receive event notifications, it MUST unsubscribe from the session. The unsubscribe request message mirrors the subscribe request message with only a single difference: the `hub.mode` MUST be equal to the string _unsubscribe_.

#### Unsubscribe Request Example

```
POST https://hub.example.com
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/x-www-form-urlencoded

hub.mode=unsubscribe&hub.topic=https%3A%2F%2Fhub.example.com%2F7jaa86kgdudewiaq0wtu&hub.events=open-patient-chart,close-patient-chart&hub.channel.type=websocket&hub.channel.endpoint=wss%3A%2F%2Fhub.example.com%2F7jaa86kgdudewiaq0wtu_1A3DF21C14514DC58B593F824D3A7ED7

```


## Event Notification

The Hub SHALL notify subscribed apps of workflow events to which the app is subscribed, as the event occurs. The notification is an UTF-8 encoded JSON object sent through the WebSocket channel.

### Event Notification Request

The notification to the subscriber SHALL include a description of the subscribed event that just occurred, an ISO 8601-2 formatted timestamp in UTC and an event identifier that is universally unique for the Hub. The timestamp MAY be used by subscribers to establish message affinity (message ordering) through the use of a message queue. The event identifier MAY be used to differentiate retried messages from user actions.

#### Event Notification Request Details

The notification's `hub.event` and `context` fields inform the subscriber of the current state of the user's session. The `hub.event` is a user workflow event, from the Event Catalog. The `context` is an array of named FHIR resources (similar to [CDS Hooks's context](https://cds-hooks.org/specification/1.0/#http-request_1) field) that describe the current content of the user's session. Each event in the [Event Catalog](#event-catalog) defines what context is expected in the notification. Hubs MAY use the [FHIR _elements parameter](https://www.hl7.org/fhir/search.html#elements) to limit the size of the data being passed while also including additional, local identifiers that are likely already in use in production implementations. Subscribers SHALL accept a full FHIR resource or the [_elements](https://www.hl7.org/fhir/search.html#elements)-limited resource as defined in the Event Catalog.

##### Event Notification Request Parameters

Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be unique for the Hub, for example a GUID.
`event` | Required | *object* | A json object describing the event. See [below](#event-notification-request-event-object-parameters).


Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session uri given in the subscription request. 
`hub.event`| Required | string | The event that triggered this notification, taken from the list of events from the subscription request.
`context` | Required | array | An array of named FHIR objects corresponding to the user's context after the given event has occurred. Common FHIR resources are: Patient, Encounter, ImagingStudy and List. The Hub SHALL only return FHIR resources that are authorized to be accessed with the existing OAuth2 access_token.

#### Event Notification Request Example

```
{
  "timestamp": "2018-01-08T01:37:05.14",
  "id": "q9v3jubddqt63n1",
  "event": {
    "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
    "hub.event": "open-patient-chart",
    "context": [
      {
        "key": "patient",
        "resource": {
          "resourceType": "Patient",
          "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
          "identifier": [
             {
               "type": {
                    "coding": [
                        {
                            "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                            "value": "MR",
                            "display": "Medication Record Number"
                         }
                        "text": "MRN"
                      ]
                  }
              }
          ]
        }
      }
    ]
  }
}
```

## Query for Current Context

In addition to receiving notification of events as they occur, a subscribing app may request the current context of a given session. The client queries the Hub's `hub.topic` url to receive the current context for the session. Event-driven context notifications should take precedence. Note that no `hub.event` is present in the response.

### Query for Current Context Example

```
GET https://hub.example.com/7jaa86kgdudewiaq0wtu 
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
```

```
{
   "timestamp":"2018-01-08T01:40:05.14",
   "id":"wYXStHqxFQyHFELh",
   "event":{
      "hub.topic":"https://hub.example.com/7jaa86kgdudewiaq0wtu",
      "context":[
         {
            "key":"patient",
            "resource":{
               "resourceType":"Patient",
               "id":"798E4MyMcpCWHab9",
               "identifier": [
                 {
                   "type": {
                        "coding": [
                            {
                                "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                                "value": "MR",
                                "display": "Medication Record Number"
                             }
                            "text": "MRN"
                          ]
                      }
                  }
              ]
            }
         },
         {
            "key":"encounter",
            "resource":{
               "resourceType":"Encounter",
               "id":"ecgXt3jVqNNpsXnNXZ3KljA3",
               "identifier":[
                  {
                     "use":"usual",
                     "system":"http://healthcare.example.org/identifiers/encounter",
                     "value":"1853"
                  }
               ]
            }
         },
         {
            "key":"study",
            "resource":{
               "resourceType":"ImagingStudy",
               "id":"8i7tbu6fby5ftfbku6fniuf",
               "uid":"urn:oid:2.16.124.113543.6003.1154777499.30246.19789.3503430045",
               "accession":{
                  "use":"usual",
                  "type":{
                     "coding":[
                        {
                           "system":"http://hl7.org/fhir/v2/0203",
                           "code":"ACSN"
                        }
                     ]
                  }
               },
               "identifier":[
                  {
                     "system":"7678",
                     "value":"185444"
                  }
               ]
            }
         }
      ]
   }
}
```


## Request Context Change

Similar to the Hub's notifications to the subscriber, the subscriber MAY request context changes with an HTTP POST to the `hub.topic` url. The Hub SHALL either accept this context change by responding with any successful HTTP status or reject it by responding with any 4xx or 5xx HTTP status. The subscriber SHALL be capable of gracefully handling a rejected context request. 

Once a requested context change is accepted, the Hub SHALL broadcast the context notification to all subscribers, including the original requestor. 

### Request Context Change Request
###### Request Context Change Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID. Following an accepted context change request, the hub MAY re-use this value in the broadcasted event notifications.
`event` | Required | *object* | A json object describing the event. See [below](#request-context-change-event-object-parameters).

###### Request Context Change Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request. 
`hub.event`| Required | string | The event that triggered this request for the subscriber, taken from the list of events from the subscription request.
`context` | Required | array | An array of named FHIR objects corresponding to the user's context after the given event has occurred. Common FHIR resources are: Patient, Encounter, ImagingStudy and List. The subscriber SHALL only include FHIR resources that are authorized to be accessed with the existing OAuth2 `access_token`.

```
POST https://hub.example.com/7jaa86kgdudewiaq0wtu HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json

{
  "timestamp": "2018-01-08T01:40:05.14",
  "id": "wYXStHqxFQyHFELh",
  "event": {
    "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
    "hub.event": "close-patient-chart",
    "context": [
      {
        "key": "patient",
        "resource": {
          "resourceType": "Patient",
          "id": "798E4MyMcpCWHab9",
          "identifier": [
             {
               "type": {
                    "coding": [
                        {
                            "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                            "value": "MR",
                            "display": "Medication Record Number"
                         }
                        "text": "MRN"
                      ]
                  }
              }
          ]
        }
      }
    ]
  }
}
```

## Sharing of Structured Information in a Case
A shared case addresses situations where multiple applications on a user’s desktop wish to exchange
structured information within a shared context.  Information is added, changed, or removed from a case frequently during this
collaboration.  Exchanged information is transitory and it is not required that the information exchanged
during the case is persisted; however, each participating application may choose to persist information per
their use cases and business logic.

![Allowed Context Hierarchies Containing a Case](../img/CaseContext.png)
*Allowed Context Hierarchies Containing a Case*

Cases exist directly inside of a patient or inside of an imaging study context.  A change or closing of a contextual subject
in which an open case is contained causes that case to be closed with all participanting applications updating their state
appropriately upon receiving the event indicating a context change (e.g. a `close-patient-chart` event).  Applications may
also choose to leave a case in which they are participating at any point in time.

Operations enabling applications to participate in a shared case session are:
* [join case](#join-case-request)
* [leave case](#leave-case-request)
* [update case](#update-case)
* [get current case content](#get-current-case-content-example-request)
* [set active resource in case](#set-active-resource-in-case-request)

### Join Case Request
A subscriber may request to join a structured information sharing case with an HTTP POST to
the `hub.topic` url. The Hub SHALL either accept this request by responding with a success HTTP status (200 - OK) 
or returning one of the below HTTP status codes if the request was not successful. The subscriber SHALL
be capable of gracefully handling a rejected join case request.

###### Request Join Case Failure - HTTP Status Codes
Code | Meaning | Condition
--- | --- | ---
400 | Bad Request | Some portion of the request was improperly formed
401 | Unauthorized | The requesting application was not recognized by the Hub
412 | Precondition Failed | The case already exists and is contained in a context other than that in the request
5xx | *per code* | Appropriate status code if a server error occurs

###### Request Join Case Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID.
`event` | Required | *object* | A json object describing the event. See [below](#request-join-case-event-object-parameters).

###### Request Join Case Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event`| Required | string | `join-case-session`<br><br>The event that triggered this request for the subscriber, taken from the list of events from the subscription request.
`case.id` | Required | string | Information sharing case identifier
`client.id` | Required | uri | Identifies the client in a non-variant manner that enables information provided by a client to be determined through multiple open and close cycles.  Only a single subscriber with a given `client.id` may join a case session. 
`context` | Required | array | An array of named FHIR objects corresponding to the user's context in which the case is to be contained. The Hub SHALL accept only the following FHIR resources: Patient, Encounter, and ImagingStudy. The subscriber SHALL only include FHIR resources that are authorized to be accessed with the existing OAuth2 `access_token`.

###### Request Join Case Example
```
POST https://hub.example.com/7jaa86kgdudewiaq0wtu HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json

{
  "timestamp": "2019-08-23T20:59:58.34",
  "id": "6213f3d3-fd34-4b3c-8474-797704f986f8",
  "event": {
    "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
    "hub.event": "join-case-session",
    "case.id": "43c6d969-23d3-4c9b-a5d2-0a1e0960ead5",
    "client.id": "app.example.client.id",
    "context": [
      {
        "key": "patient",
        "resource": {
          "resourceType": "Patient",
          "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
          "identifier": [
            {
              "type": {
                "coding": [
                  {
                    "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                    "value": "MR",
                    "display": "Medication Record Number"
                  }
                ],
                "text": "MRN"
              },
              "value":  "338742",
              "assigner":  "Riverside"
            }
          ]
        }
      },
      {
        "key": "encounter",
        "resource": {
          "resourceType": "Encounter",
          "id": "ecgXt3jVqNNpsXnNXZ3KljA3",
          "identifier": [
            {
              "value": "ipe1389914"
            }
          ]
        }
      }
    ]
  }
}
```

The `case.id` is an identifier which is known by all applications who wish to participate in the case sharing session.  This
could be the same as the `hub.topic` (if applications unsubscribe from a topicfor each case), it may be an id handed out by some controlling
application (e.g. a worklist component), or it may be some information natural to the case such as an accession number.

The `client.id` is a uri which should be invariant across the entire lifecycle of a given case.  It is not intended to be specific
to a given workstation, rather for a specific application across all relevant workstations.  For example, there may be 10
workstations on which Application A is installed.  The `client.id` for Application A should be the same for cases opened on all
of these workstations.  The `client.id` should not be used for special logic by the Hub or any other application – it should be used
only for the purposes of limiting modification or removal by applications other than the contributor of the specific resource.

##### Response Join Case Parameters
Upon successful completion of the request the Hub SHALL return the following response in addition to
returning the HTTP 200 status.

Field | Optionality | Type | Description
--- | --- | --- | ---
`id` | Required | string | Event identifier which was uniquely generated by the requesting subscriber.
`hub.event` | Required | string | `join-case-session`<br><br>The event that triggered this request from the subscriber.
`case.version` | Required	| string | The current version of the case at the time the requesting application is joining the case.  This version is used to ensure any subsequent request from subscribers is using the current version of the case and shall be a new version after each operation that alters content of the case.

If the join request fails, the Hub MAY respond with an operational outcome resource in addition to returning
the appropriate HTTP failure status. The following operation outcome issue type and operation outcome codes
are recommended:

HTTP Status | Issue Type | Operation Outcome | Condition
--- | --- | --- | ---
400 | structure | MSG_BAD_SYNTAX | Some portion of the request was improperly formed
401 | forbidden | MSG_AUTH_REQUIRED | The requesting application was not recognized by the Hub
412 | conflict | MSG_WRONG_NS | The case already exists and is contained in a context other than that in the request
5xx | transient | *not applicable* | Appropriate status code if a server error occurs

### Leave Case Request
A subscriber may request to leave a structured information sharing case in which they are already participating
with an HTTP POST to the `hub.topic` url.  The Hub SHALL either acknowledge the leave request was honored
successfully by returning a success HTTP status (200 - OK) or returning one of the below HTTP status codes if the request
was not successful.  The subscriber SHALL be able to appropriately handle a failed leave case request.

###### Request Leave Case Failure - HTTP Status Codes
Code | Meaning | Condition
--- | --- | ---
400 | Bad Request | Some portion of the request was improperly formed
401 | Unauthorized | The requesting application was not recognized by the Hub
410 | Gone | The case was not found
412 | Precondition Failed | The application is not subscribed to the `hub.topic` or the the application has not joined the case
5xx | *per code* | Appropriate status code if a server error occurs

###### Request Leave Case Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID.
`event` | Required | *object* | A json object describing the event. See [below](#request-leave-case-event-object-parameters).

###### Request Leave Case Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event`| Required | string | `leave-case-session`<br><br>The event that triggered this request for the subscriber, taken from the list of events from the subscription request.
`case.id` | Required | string | Information sharing case identifier

###### Request Leave Case Example
```
POST https://hub.example.com/7jaa86kgdudewiaq0wtu HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json

{
  "timestamp": "2019-08-23T21:15:48.10",
  "id": "fd42787c-0366-4789-bc8e-f0a8638d24a7",
  "event": {
    "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
    "hub.event": "leave-case-session",
    "case.id": "43c6d969-23d3-4c9b-a5d2-0a1e0960ead5"
  }
}
```

If the leave request fails, the Hub MAY respond with an operational outcome resource in addition to returning
the appropriate HTTP failure status.  The following operation outcome issue type and operation outcome codes
are recommended:

HTTP Status | Issue Type | Operation Outcome | Condition
--- | --- | --- | ---
400 | structure | MSG_BAD_SYNTAX | Some portion of the request was improperly formed
401 | forbidden | MSG_AUTH_REQUIRED | The application was not granted an authorization scope required to add the resource type of one of the FHIR resources being added
410 | not-found | MSG_NO_EXIST | The case was not found
412 | unknown | MSG_AUTH_REQUIRED | The application is not subscribed to the `hub.topic` or the the application has not joined the case
5xx | transient | *not applicable* | Appropriate status code if a server error occurs

### Update Case Request
A subscriber may request to add, remove or change one or more FHIR resources to a structured information sharing case in
which they are participating with an HTTP POST to the `hub.topic` url.  The Hub SHALL either acknowledge
that the request was honored successfully by returning a success HTTP status (200 - Accepted) or
returning one of the below HTTP status codes if the request was not successful.  The Hub SHALL perform all
additions atomically and all resources in the request must be added successfully or the Hub MUST fail
the entire request and return one of the below errors.  The subscriber SHALL be able to appropriately
handle a failed add request.

###### Request Update Case Failure - HTTP Status Codes
Code | Meaning | Condition
--- | --- | ---
400 | Bad Request | Some portion of the request was improperly formed
401 | Unauthorized | The application was not granted an authorization scope required to add the resource type of one of the FHIR resources being added
406 | Not Acceptable | One of the FHIR resources being added has an id of a resource already present in the structured information sharing case
410 | Gone | The case was not found
412 | Precondition Failed | The application is not subscribed to the `hub.topic` or the the application has not joined the case
422 | Unprocessable Entity | One of the FHIR resources being added was structurally malformed
428 | Precondition Required | The case.version of the submitted request is not the current version of the case held by the Hub
5xx | *per code* | Appropriate status code if a server error occurs

###### Request Update Case Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID.
`event` | Required | *object* | A json object describing the event. See [below](#request-add-resource-to-case-event-object-parameters).

###### Request Update CaseEvent Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event`| Required | string | `add-resource-to-case`<br><br>The event that triggered this request for the subscriber, taken from the list of events from the subscription request.
`case.id` | Required | string | Information sharing case identifier
`case.version` | Required | string | Version of the case upon which the structured information held by the subscriber was based.
`client.id` | Required | uri | Identifies the client in a non-variant manner that enables information provided by a client to be determined through multiple open and close cycles.  Only a single subscriber with a given client.id may join a session. 
`content` | Required | Bundle |  A (https://www.hl7.org/fhir/bundle.html)FHIR Bundle The Hub SHALL only accept FHIR resources that are authorized to be accessed with the existing OAuth2 `access_token`.

###### Request Update Case Example
```
POST https://hub.example.com/7jaa86kgdudewiaq0wtu HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json

{
  "timestamp": "2019-08-23T20:59:58.34",
  "id": "1f5164e2-817a-4d55-8969-8b20c9c8648b",
  "event": {
    "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
    "hub.event": "update-case",
    "case.id": "43c6d969-23d3-4c9b-a5d2-0a1e0960ead5",
    "case.version": "6ab3619c-3551-4c9b-b201-17643ebc19645",
    "client.id": "app_example_client_id",
    "content": [
      {
    "resourceType": "Bundle",
    "id": "8i7tbu6fby5ftfbku6fniuf",
    "type" : "collection",
    "entry": [
      {
        "request": {
          "method" : "PUT"
        }
        "resource" : {
          "resourceType": "Observation",
          "id": "bmd",
          "text": {
            "status": "generated",
          },
          "status": "final",
          "code": {
            "coding": [
              {
                "system": "http://loinc.org",
                "code": "24701-5",
                "display": "Femur DXA Bone density"
              }
            ],
            "text": "BMD - Left Femur"
          },
          "subject": {
            "reference": "Patient/pat2"
          },
          "performer": [
            {
              "reference": "Organization/1832473e-2fe0-452d-abe9-3cdb9879522f",
              "display": "Acme Imaging Diagnostics"
            }
          ],
          "valueQuantity": {
            "value": 0.887,
            "unit": "g/cm²",
            "system": "http://unitsofmeasure.org",
            "code": "g/cm-2"
          },
          "bodySite": {
            "coding": [
              {
                "system": "http://snomed.info/sct",
                "code": "71341001:272741003=7771000"
              }
            ],
            "text": "Left Femur"
          }
        }
      }
    ]
  }
}
```

The Bundle Resource holds a list of 1 to n resources which are requested to be added (POST), removed (DELETE) or changed (PUT) in the case. Each
resource MAY add the `client.id` to the source attribute of the resource’s meta attribute.

In the tag attribute of a resource’s meta attribute an application MAY specify one or more protection
levels for the resource (in the above example, the Observation resource contains an update restriction
while the ImagingStudy resource does not specify any protection level).  The following protection levels
are available in the http://fhircast.org/exchange/protection code set:

Code | Meaning
--- | ---
`restrict.update` | No application other than the application that added the resource to the case may update any content of the resource
`restrict.removal`| No application other than the application that added the resource to the case may remove the resource from the case

##### Response Update CaseParameters
Upon successful completion of the request the Hub SHALL return the following response in addition to
returning the HTTP 200 Accepted status.

Field | Optionality | Type | Description
--- | --- | --- | ---
`id` | Required | string | Event identifier which was uniquely generated by the requesting subscriber.
`hub.event` | Required | string | `add-resource-to-case`<br><br>The event that triggered this request from the subscriber.
`case.version` | Required	| string | A new unique version of the case which is generated by the Hub.  This version is used to ensure any subsequent request from subscribers is using the current version of the case and shall be a new version after each operation that alters content of the case.

If the request fails, the Hub MAY respond with an operational outcome resource in addition to returning
the appropriate HTTP failure status.  The following operation outcome issue type and operation outcome codes
are recommended:

HTTP Status | Issue Type | Operation Outcome | Condition
--- | --- | --- | ---
400 | structure | MSG_BAD_SYNTAX | Some portion of the request was improperly formed
401 | forbidden | MSG_AUTH_REQUIRED | The application was not granted an authorization scope required to add the resource type of one of the FHIR resources being added
406 | duplicate | MSG_DUPLICATE_ID | One of the FHIR resources which was requested to be added has an id of a resource already present in the structured information sharing case
410 | not-found | MSG_NO_EXIST | The case was not found
412 | unknown | MSG_AUTH_REQUIRED | The application is not subscribed to the `hub.topic` or the the application has not joined the case
422 | structure |  MSG_CANT_PARSE_CONTENT | One of the FHIR resources which was requested to be added was structurally malformed
428 | conflict | MSG_VERSION_AWARE_CONFLICT | The `case.version` of the submitted request is not the current version of the case held by the Hub
5xx | transient | *not applicable* | Appropriate status code if a server error occurs

##### Update CaseNotification Parameters
The Hub SHALL send the 'update-case' event (see in Event Catalog) to all applications (including the submitting client) participating in the case if the request was successfully completed.
The Hub SHALL not send any notification to other applications if the request fails.

### Get Current Case Content Request
A subscribing application may request the current content of a shared case in which they are participating with an HTTP GET to the `hub.topic` url.  The client
queries the case by the Hub's `hub.topic` url with the `case.id` as a parameter.  Note that no `hub.event` is present in the response.  If the query is successful
the Hub shall respond by returning a success HTTP status (200 - OK), HTTP status (204 - No Content), or returning one of the below HTTP status codes if the
request was not successful.  The subscriber SHALL be able to appropriately handle a failed request.

###### Get Current Case Content Failure - HTTP Status Codes
Code | Meaning | Condition
--- | --- | ---
401 | Unauthorized | The application was not granted an authorization scope required to get the case's current context
410 | Gone | The case was not found
412 | Precondition Failed | The application is not subscribed to the `hub.topic` or the the application has not joined the case
5xx | *per code* | Appropriate status code if a server error occurs

###### Get Current Case Content Example Request

```
GET https://hub.example.com/7jaa86kgdudewiaq0wtu?case.id=43c6d969-23d3-4c9b-a5d2-0a1e0960ead5 HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json
```
###### Get Current Case Content Example Response
```
{
    "resourceType": "Bundle",
    "id": "8i7tbu6fby5ftfbku6fniuf",
    "type" : "collection",
    "entry": [
      {
        "request": {
          "method" : "PUT"
        }
        "resource" : {
          "resourceType": "Observation",
          "id": "bmd",
          "text": {
            "status": "generated",
          },
          "status": "final",
          "code": {
            "coding": [
              {
                "system": "http://loinc.org",
                "code": "24701-5",
                "display": "Femur DXA Bone density"
              }
            ],
            "text": "BMD - Left Femur"
          },
          "subject": {
            "reference": "Patient/pat2"
          },
          "performer": [
            {
              "reference": "Organization/1832473e-2fe0-452d-abe9-3cdb9879522f",
              "display": "Acme Imaging Diagnostics"
            }
          ],
          "valueQuantity": {
            "value": 0.887,
            "unit": "g/cm²",
            "system": "http://unitsofmeasure.org",
            "code": "g/cm-2"
          },
          "bodySite": {
            "coding": [
              {
                "system": "http://snomed.info/sct",
                "code": "71341001:272741003=7771000"
              }
            ],
            "text": "Left Femur"
          }
        }
      }
    ]
```

The `content` field holds a Bundle of resources which are present in the case.

If the get content request fails, the Hub MAY respond with an operational outcome resource in addition to returning
the appropriate HTTP failure status.  The following operation outcome issue type and operation outcome codes
are recommended:

HTTP Status | Issue Type | Operation Outcome | Condition
--- | --- | --- | ---
401 | forbidden | MSG_AUTH_REQUIRED | The application was not granted an authorization scope required to get the current content of the case
410 | not-found | MSG_NO_EXIST | The case was not found
412 | unknown | MSG_AUTH_REQUIRED | The application is not subscribed to the `hub.topic` or the the application has not joined the case
5xx | transient | *not applicable* | Appropriate status code if a server error occurs

### Set Active Resource in Case Request
A subscriber may request a resource be noted as being active or inactive in the case in which they are participating with an
HTTP POST to the `hub.topic` url.  The Hub SHALL either acknowledge that the add request was honored successfully by returning a success 
HTTP status (200 - OK) or returning one of the below HTTP status codes if the request was not successful.  The subscriber SHALL be able
to appropriately handle a failed activate/inactivate request.

Noting a resource as active (or inactive) allows applications to react to a user’s selection of structured information in another application.
Hence an application may want to set (or clear) an active resource with others reacting to the selection if appropriate.  For example,
a reporting system may indicate that the user has selected a particular observation with a measurement value and an imaging application
which created the measurement may wish to change its user display such that the image from which the measurement was acquired is visible.

###### Set Active Resource in Case Failure - HTTP Status Codes
Code | Meaning | Condition
--- | --- | ---
400 | Bad Request | Some portion of the request was improperly formed
401 | Unauthorized | The application was not granted an authorization scope required to note as active the resource type of the FHIR resource requested to be made active
404	| Not Found	| The resource which is requested to be noted as active or inactive was not found in the case
410 | Gone | The case was not found
412 | Precondition Failed | The application is not subscribed to the `hub.topic` or the the application has not joined the case
428 | Precondition Required | The case.version of the submitted request is not the current version of the case held by the Hub
5xx | *per code* | Appropriate status code if a server error occurs

###### Set Active Resource in Case Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID.
`event` | Required | *object* | A json object describing the event. See [below](#set-active-resource-in-case-event-object-parameters).

###### Set Active Resource in Case Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event`| Required | string | `set-active-resource-in-case`<br><br>The event that triggered this request for the subscriber, taken from the list of events from the subscription request.
`case.id` | Required | string | Information sharing case identifier
`case.version` | Required | string | Version of the case upon which the structured information held by the subscriber was based.
`resource.id` | Required | string | FHIR resource id that should be noted as active or inactive in the case
`resource.state` | Required | string | Indicates whether the indicated resource should be noted as active or inactive - the only permitted string values are `active` and `inactive`

##### Response Set Active Resource in Case Parameters
Upon successful completion of the request the Hub SHALL return the following response in addition to returning the HTTP 200 status.

Field | Optionality | Type | Description
--- | --- | --- | ---
`id` | Required | string | Event identifier which was uniquely generated by the requesting subscriber.
`hub.event` | Required | string | `set-active-resource-in-case`<br><br>The event that triggered this request from the subscriber.
`case.version` | Required	| string | A new unique version of the case which is generated by the Hub.  This version is used to ensure any subsequent request from subscribers is using the current version of the case and shall be a new version after each operation that alters content of the case.

If the set active request fails, the Hub MAY respond with an operational outcome resource in addition to returning the appropriate HTTP failure status.
The following operation outcome issue type and operation outcome codes are recommended:

HTTP Status | Issue Type | Operation Outcome | Condition
--- | --- | --- | ---
400 | structure | MSG_BAD_SYNTAX | Some portion of the request was improperly formed
401 | forbidden | MSG_AUTH_REQUIRED | The application was not granted an authorization scope required to note as active the resource type of the FHIR resource requested to be made active
404 | not-found | MSG_NO_EXIST | The resource which is requested to be removed was not found in the case
410 | not-found | MSG_NO_EXIST | The case was not found
412 | unknown | MSG_AUTH_REQUIRED | The application is not subscribed to the `hub.topic` or the the application has not joined the case
428 | conflict | MSG_VERSION_AWARE_CONFLICT | The `case.version` of the submitted request is not the current version of the case held by the Hub
5xx | transient | *not applicable* | Appropriate status code if a server error occurs

##### Set Active Resource in Case Notification Parameters
The Hub SHALL send the following event to all other applications participating in the case if the request was successfully completed.  The requesting application
SHALL not be sent this event.  The Hub SHALL not send any notification to other applications if the request fails.

Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | string | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy.
`id` | Required | string | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID. Following an accepted context change request, the hub MAY re-use this value in the broadcasted event notifications.
`event` | Required | object | A json object describing the event, see [below](#set-active-resource-in-case-notification-event-object-parameters).

##### Set Active Resource in Case Notification Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event` | Required | string | `remove-resource-from-case`<br>The event that triggered this request from the subscriber, taken from the list of events from the subscription request.
`case.id` | Required | string | Information sharing case identifier
`case.version` | Required | string | New unique version of the case which is generated by the Hub after the set active request was successful
`resource.id` | Required | string | FHIR resource id that should be noted as active or inactive in the case
`resource.state` | Required | string | Indicates whether the indicated resource should be noted as active or inactive - the only permitted string values are `active` and `inactive`

## Event Notification Errors

If the subscriber cannot follow the context of the event, for instance due to an error or a deliberate choice to not follow a context, the subscriber MAY respond with a 'sync-error' event. The Hub MAY use these events to track synchronization state and MAY also forward these events to other subscribers of the same topic.

### Event Notification Error Example

```
POST https://hub.example.com/7jaa86kgdudewiaq0wtu HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json

{
  "timestamp": "2018-01-08T01:37:05.14",
  "id": "q9v3jubddqt63n1",
  "event": {
    "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
    "hub.event": "sync-error",
    "context": [
      {
        "key": "operationoutcome",
        "resource": {
          "resourceType": "OperationOutcome",
          "issue": [
            {
              "severity": "warning",
              "code": "processing",
              "diagnostics": "AppId3456 failed to follow context"
            }
          ]
        }
      }
    ]
  }
}
```

## Event Catalog
Each event definition in the catalog, below, specifies a single event name, a description of the event, and the  required or optional contextual information associated with the event. Alongside the event name, the contextual information is used by the subscriber.

FHIR is the interoperable data model used by FHIRcast. The fields within `context` are subsets of FHIR resources. Hubs MUST send the results of FHIR reads in the context, as specified below. For example, when the `open-image-study` event occurs, the notification sent to a subscriber MUST include the ImagingStudy FHIR resource. Hubs SHOULD send the results of an ImagingStudy FHIR read using the *_elements* query parameter, like so:  `ImagingStudy/{id}?_elements=identifier,accession` and in accordance with the [FHIR specification](https://www.hl7.org/fhir/search.html#elements). 

A FHIR server may not support the *_elements* query parameter; a subscriber MUST gracefully handle receiving a full FHIR resource in the context of a notification.

The name of the event SHOULD succinctly and clearly describe the activity or event. Event names are unique so event creators SHOULD take care to ensure newly proposed events do not conflict with an existing event name. Event creators SHALL name their event with reverse domain notation (e.g. `org.example.patient-transmogrify`) if the event is specific to an organization. Reverse domain notation SHALL not be used by a standard event catalog.

### open-patient-chart
#### Description: 
User opened patient's medical record. 
#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
           {
             "type": {
                  "coding": [
                      {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "value": "MR",
                          "display": "Medication Record Number"
                       }
                      "text": "MRN"
                    ]
                }
            }
        ]
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Required| `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context.
`encounter` | Optional | `Encounter/{id}?_elements=identifier` | FHIR Encounter resource in context in the newly opened patient's chart.


### switch-patient-chart

#### Description: 
User changed from one open patient's medical record to another previously opened patient's medical record. The context documents the patient whose record is currently open.
#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
           {
             "type": {
                  "coding": [
                      {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "value": "MR",
                          "display": "Medication Record Number"
                       }
                      "text": "MRN"
                    ]
                }
            }
        ]
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Required |  `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context..
`encounter` | Optional | `Encounter/{id}?_elements=identifier` | FHIR Encounter resource in context in the newly opened patient's chart.


### close-patient-chart

#### Description: User closed patient's medical record. 

#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
           {
             "type": {
                  "coding": [
                      {
                          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                          "value": "MR",
                          "display": "Medication Record Number"
                       }
                      "text": "MRN"
                    ]
                }
            }
        ]
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Required |  `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context..
`encounter` | Optional | `Encounter/{id}?_elements=identifier` | FHIR Encounter resource in context in the newly opened patient's chart.

### open-imaging-study
#### Description: User opened record of imaging study.
#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
          {
            "system": "urn:oid:1.2.840.114350",
            "value": "185444"
          },
          {
            "system": "urn:oid:1.2.840.114350.1.13.861.1.7.5.737384.27000",
            "value": "2667"
          }
        ]
      }
    },
    {
      "key": "study",
      "resource": {
        "resourceType": "ImagingStudy",
        "id": "8i7tbu6fby5ftfbku6fniuf",
        "uid": "urn:oid:2.16.124.113543.6003.1154777499.30246.19789.3503430045",
        "identifier": [
          {
            "system": "7678",
            "value": "185444"
          }
        ],
        "patient": {
          "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
        }
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Optional| `Patient/{id}?_elements=identifier`| FHIR Patient resource describing the patient whose chart is currently in context.
`study` | Required | `ImagingStudy/{id}?_elements=identifier,accession` | FHIR ImagingStudy resource in context. Note that in addition to the request identifier and accession elements, the DICOM uid and FHIR patient reference are included because they're required by the FHIR specification. 

### switch-imaging-study
#### Description: User changed from one open imaging study to another previously opened imaging study. The context documents the study, and optionally patient, for the currently open record.
#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
          {
            "system": "urn:oid:1.2.840.114350",
            "value": "185444"
          },
          {
            "system": "urn:oid:1.2.840.114350.1.13.861.1.7.5.737384.27000",
            "value": "2667"
          }
        ]
      }
    },
    {
      "key": "study",
      "resource": {
        "resourceType": "ImagingStudy",
        "id": "8i7tbu6fby5ftfbku6fniuf",
        "uid": "urn:oid:2.16.124.113543.6003.1154777499.30246.19789.3503430045",
        "identifier": [
          {
            "system": "7678",
            "value": "185444"
          }
        ],
        "patient": {
          "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
        }
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Optional| `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context.
`study` | Required | `ImagingStudy/{id}?_elements=identifier,accession` | FHIR ImagingStudy resource in context. Note that in addition to the request identifier and accession elements, the DICOM uid and FHIR patient reference are included because they're required by the FHIR specification. 

### close-imaging-study

#### Description: User closed imaging study.

#### Example: 
```
{
  "context": [
    {
      "key": "patient",
      "resource": {
        "resourceType": "Patient",
        "id": "ewUbXT9RWEbSj5wPEdgRaBw3",
        "identifier": [
          {
            "system": "urn:oid:1.2.840.114350",
            "value": "185444"
          },
          {
            "system": "urn:oid:1.2.840.114350.1.13.861.1.7.5.737384.27000",
            "value": "2667"
          }
        ]
      }
    },
    {
      "key": "study",
      "resource": {
        "resourceType": "ImagingStudy",
        "id": "8i7tbu6fby5ftfbku6fniuf",
        "uid": "urn:oid:2.16.124.113543.6003.1154777499.30246.19789.3503430045",
        "identifier": [
          {
            "system": "7678",
            "value": "185444"
          }
        ],
        "patient": {
          "reference": "Patient/ewUbXT9RWEbSj5wPEdgRaBw3"
        }
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`patient` | Optional| `Patient/{id}?_elements=identifier` | FHIR Patient resource describing the patient whose chart is currently in context.
`study` | Required | `ImagingStudy/{id}?_elements=identifier,accession` | FHIR ImagingStudy resource in context. Note that in addition to the request identifier and accession elements, the DICOM uid and FHIR patient reference are included because they're required by the FHIR specification. 

### update-case
#### Description: User added, removed or changed one or more resources part of the currently opened context.
#### Example:
```
{
  "key": "bundle",
  "resource": {
    "resourceType": "Bundle",
    "id": "8i7tbu6fby5ftfbku6fniuf",
    "type" : "collection",
    "entry": [
      {
        "request": {
          "method" : "PUT"
        }
        "resource" : {
          "resourceType": "Observation",
          "id": "bmd",
          "text": {
            "status": "generated",
          },
          "status": "final",
          "code": {
            "coding": [
              {
                "system": "http://loinc.org",
                "code": "24701-5",
                "display": "Femur DXA Bone density"
              }
            ],
            "text": "BMD - Left Femur"
          },
          "subject": {
            "reference": "Patient/pat2"
          },
          "performer": [
            {
              "reference": "Organization/1832473e-2fe0-452d-abe9-3cdb9879522f",
              "display": "Acme Imaging Diagnostics"
            }
          ],
          "valueQuantity": {
            "value": 0.887,
            "unit": "g/cm²",
            "system": "http://unitsofmeasure.org",
            "code": "g/cm-2"
          },
          "bodySite": {
            "coding": [
              {
                "system": "http://snomed.info/sct",
                "code": "71341001:272741003=7771000"
              }
            ],
            "text": "Left Femur"
          }
        }
      }
    ]
  }
}
```

The context object shall contain exactly one Resource of type [*Bundle*](https://www.hl7.org/fhir/bundle.html) with all affected resource(s) in its `entry` field. Each `entry` MUST carry a `request` and a `resource` field with the following interpretation:

* `entry.request.method` type of operation the user performed on the resource
    * PUT = User created the Resource
    * DELETE = User deleted the Resource
    * POST = User updated the Resource
* `entry.request.resource` the *complete* resource object for PUT and POST operations
or the resource object including a unique identifier for DELETE operations

### user-logout
#### Description: User gracefully exited the application.
#### Example: 
{
}


No Context 

### user-hibernate
#### Description: User temporarily suspended her session. The user's session will eventually resume.
#### Example: 
{
}

No Context

### sync-error
#### Description: A synchronization error has been detected. Inform subscribed clients.
#### Example: 
```
{
  "context": [
    {
      "key": "operationoutcome",
      "resource": {
        "resourceType": "OperationOutcome",
        "issue": [
          {
            "severity": "warning",
            "code": "processing",
            "diagnostics": "AppId3456 failed to follow context"
          }
        ]
      }
    }
  ]
}
```

Context | Optionality | FHIR operation to generate context|  Description
--- | --- | --- | ---
`operationoutcome` | Optional |  `OperationOutcome` | FHIR resource describing an outcome of an unsuccessful system action..
