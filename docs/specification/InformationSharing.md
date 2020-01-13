## Sharing of Structured Information in a Context Container
Multiple applications on a user’s desktop may wish to exchange structured information within a shared context.
Information is added, changed, or removed from a case frequently during such collaborations.  Exchanged information
is transitory and it is not required that the information exchanged during the collaboration is persisted. However,
as required by their use cases, each participating application may choose to persist information in their own structures
which may or may not be expressed as a FHIR resource. Even if stored in the form of a FHIR resource, the resource may or may
not be stored in a system fronted by a FHIR server.

The contextual subjects of Patient, Imaging Study, or a Diagnostic Report may serve as a container of structured information.
Any type of FHIR resource may be added as structured information to the containing contextual subject.  Applications may choose
to participate in information sharing sessions by listening to change-content events and add new information using the update-content
operation. A change or closing of a contextual subject in which content resources are enclosed clears all content.

![Allowed Context Containers for Information Sharing](../img/ContextContainers.png)
*Allowed Context Containers*

It should be noted that *content* and *context* are distinct concepts. A *context* is a circumstance on which multiple
desktop applications agree to synchronize.  *Content* is information which multiple applications agree to share inside of a
given *context*.  While most interactions are the same between *contextual* subjects and their contained *content*, there are some
differences in associated operations and a few additional responsibilities are required of the Hub. Additionally, a given Hub may
or may not support the sharing of structured information. If a Hub does not support sharing of structured information requesting
applications should expect and handle the standard not implemented HTTP error status (501 - Not Implemented) when requesting
one of the structured information sharing operations be performed.

Finally, context resources that act as content containers (i.e. Patient, Imaging Study, and Diagnostic Report) may contain content
of the same or other potential context resources that may be content containers.  For example, a Diagnostic Report acting as a
content container could contain an Imaging Study as a resource in its content.  In another example, an Imaging Study acting as a
content container could contain another Imaging Study in its content - perhaps as a way of noting that the contained Imaging Study
is a prior Imaging Study to the Imaging Study acting as the content container. No validation of the semantic appropriateness of
content is performed by the Hub.

Operations enabling applications to participate sharing structured information are:

* [update content](#update-content-request)
* [get current content](#get-current-content-request)
* [set active resource](#set-active-resource-request)

Responsibilities of a Hub supporting structured information sharing is provided [below](#hub-responsibilities-to-support-sharing-of-structured-information).

### Example Use Case
A frequent scenario which illustrates a usage of content sharing involves an image reading application, a reporting application, and an advanced quantification application.  The imaging reading and reporting applications may be active subscribers to a FHIRcast topic.  They would commonly react to a setting or change of the FHIRcast topic's context.  Inside of this context (e.g. an imaging study) they exchange structured information (i.e. content), while the user is interacting with one or the other application, using the `update content` request.  At some point the imaging application (automatically or through user interaction) may determine that an advanced quantification application should be used and launch the application including the appropriate FHIRcast topic.  The advanced quantification application may determine it would like to retrieve any content available in the current FHIRcast context by using the `get current content` request as well as contributing content using the `update content` request.  Finally, the user may select a measurement in the reporting application which could then use the `set active resource` request to which the imaging application responds by navigating to the image on which the measurement was acquired.

### Update Content Request
A subscriber may request to add, remove, or change one or more FHIR resources to the content of a context container with an HTTP
POST to the `hub.topic` url. The Hub SHALL either acknowledge that the request was honored successfully by returning a success
HTTP status (200 - OK) or returning an [HTTP error status](#update-content-request-failure) if the request fails. The Hub
SHALL perform all requested changes atomically and all requested changes must either succeed or the Hub MUST fail the entire
request. The subscriber SHALL be able to appropriately handle a failed update request.

###### Request Update Content Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID.
`event` | Required | *object* | A json object describing the event. See [below](#request-update-content-event-object-parameters).

###### Request Update Content Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event`| Required | string | `update-content`<br><br>The event that triggered this request for the subscriber, taken from the list of events from the subscription request.
`container.id` | Required | string | Resource id of the contextual subject in which this content resides
`content.timestamp` | Required | string | ISO 8601-2 timestamp in UTC provided by the Hub when it last updated content
`client.id` | Optional | uri | Identifies the client in a non-variant manner that enables information provided by a client to be determined through multiple open and close cycles.  Only a single subscriber with a given client.id may join a session. 
`content` | Required | Bundle |  A [FHIR Bundle resource](http://www.hl7.org/fhir/bundle.html) containing the requested content changes.  The Hub SHALL only accept FHIR resources that are authorized to be accessed with the existing OAuth2 `access_token`.

###### Request Update Content Example
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
    "hub.event": "update-content",
    "container.id": "43c6d969-23d3-4c9b-a5d2-0a1e0960ead5",
    "content.timestamp": "2019-08-23T21:05:22.59",
    "client.id": "app_example_client_id",
    "content": [
      {
        "resourceType": "Bundle",
        "id": "8i7tbu6fby5ftfbku6fniuf",
        "type": "collection",
        "entry": [
          {
            "request": {
              "method": "PUT"
            },
            "resource": {
              "resourceType": "Observation",
              "id": "bone-density",
              "meta": {
                "source": "app_example_client_id"
              },
              "text": {
                "status": "generated"
              },
              "partOf": [
                {
                  "reference": "ImagingStudy/f65fd7e0-14d5-4202-aa17-249957fe2803"
                }
              ],
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
                "reference": "Patient/160f4ff2-6a32-45e4-810d-829d7d578442"
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
          },
          {
            "request": {
              "method": "POST"
            },
            "resource": {
              "resourceType": "Observation",
              "id": "head-circumference",
              "meta": {
                "source": "app_example_client_id",
                "tag": [
                  {
                    "coding": [
                      {
                        "system": "http://fhircast.org/exchange/protection",
                        "code": "restrict.update"
                      }
                    ]
                  }
                ]
              },
              "partOf": [
                {
                  "reference": "ImagingStudy/f65fd7e0-14d5-4202-aa17-249957fe2803"
                }
              ],
              "status": "preliminary",
              "code": {
                "coding": [
                  {
                    "system": "http://loinc.org",
                    "code": "9843-4",
                    "display": "Head Occipital-frontal circumference"
                  }
                ],
                "text": "Head Circumference"
              },
              "subject": {
                "reference": "Patient/160f4ff2-6a32-45e4-810d-829d7d578442"
              },
              "valueQuantity": {
                "value": 51.2,
                "unit": "cm",
                "system": "http://unitsofmeasure.org",
                "code": "cm"
              }
            }
          }
        ]
      }
    ]
  }
}
```

The Bundle Resource holds a list of 1 to n resources which are requested to be added (POST), removed (DELETE) or changed (PUT) in the
content as specified in the request/method attribute of each resource in the bundle.

Each resource MAY contain the `client.id` to the source attribute of the resource’s meta attribute.  In the above example, both
observation resources specify a client id.

###### Resource Protection Levels
In the tag attribute of a resource’s meta attribute an application MAY specify one or more protection levels for the resource.  In
the above example, the head circumference observation resource contains an update restriction while the bone density
observation resource does not specify any protection level.  The following protection levels are available in the
http://fhircast.org/exchange/protection code set:

Code | Meaning
--- | ---
`restrict.update` | No application other than the application that added the resource to the case may update any content of the resource
`restrict.removal`| No application other than the application that added the resource to the case may remove the resource from the case

If a resource does not contain a value in the `source` attribute of the resource's meta attribute, then any protection level is included
in the resource is disregarded.

##### Response Update Content Parameters
Upon successful completion of the request the Hub SHALL return the following response in addition to
returning the HTTP 200 Accepted status.

Field | Optionality | Type | Description
--- | --- | --- | ---
`id` | Required | string | Event identifier which was uniquely generated by the requesting subscriber.
`hub.event` | Required | string | `update-content`<br><br>The event that triggered this request from the subscriber.
`content.timestamp` | Required | string | Timestamp generated by the Hub when it updated content based on this request.  This timestamp is used to ensure subsequent request from any subscriber is using the current version of the content.

##### Update Content Request Failure
If the update content request fails, the Hub SHALL respond with one of the below HTTP error status codes corresponding with the described failure
condition.

###### Request Update Content Failure - HTTP Error Status Codes
Code | Meaning | Condition
--- | --- | ---
400 | Bad Request | Some portion of the request was improperly formed or a duplicate `client.id` was provided in the request
401 | Unauthorized | The application was not granted an authorization scope required to add, update, or remove the resource type of one of the FHIR resources in the request
403	| Forbidden	| Update or removal of one of the resources in the request is not permitted by any application except the creator of the resource (see [resource protection levels](#resource-protection-levels))
404	| Not Found	| One of the FHIR resources which is requested to be updated or removed was not found
409 | Conflict | One of the FHIR resources which is requested to be added has an id of a resource already present
410 | Gone | The containing context resource (`container.id`) provided in the request was not found
412 | Precondition Failed | The application is not subscribed to the `hub.topic`
422 | Unprocessable Entity | One of the FHIR resources in the request was structurally malformed
428 | Precondition Required | The `content.timestamp` in the submitted request is not the current timestamp of the content held by the Hub
5xx | *per code* | Appropriate status code if a server error occurs

###### Request Update Content Failure - Operational Outome Response Recommendations
If the update content request fails, the Hub MAY respond with a [FHIR Operation Outcome resource](http://www.hl7.org/fhir/operationoutcome.html)
in addition to returning the appropriate HTTP error status code.  The operation outcome `severity` SHALL be 'error' while
the following `code` and `details` values are recommended:

HTTP Status | Code | Details
--- | --- | ---
400 | structure | MSG_BAD_SYNTAX 
401 | forbidden | MSG_AUTH_REQUIRED
403 | lock-error | MSG_RESOURCE_NOT_ALLOWED
404 | not-found | MSG_NO_EXIST
409 | duplicate | MSG_DUPLICATE_ID
410 | not-found | MSG_NO_EXIST
412 | unknown | MSG_AUTH_REQUIRED
422 | structure |  MSG_CANT_PARSE_CONTENT
428 | conflict | MSG_VERSION_AWARE_CONFLICT
5xx | transient | *not applicable*

In order to assist the requestor in identifying the cause of the request's failure, it is highly recommended that in the case of an update content failure with the error status of 403, 404, or 409 that an operation outcome resource be returned containing the id(s) of resources which caused the request to fail.  The id(s) should be contained in OperationOutcome.issue.expression.

##### Update Content Event
If the update content request succeeds, the Hub SHALL send the following event to all other subscribers. The Hub SHALL NOT send this event
to the requesting application. The Hub SHALL NOT send any notification to other subscribers if the request fails.

###### Update Content Notification Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event` | Required | string | `update-content`<br>The event that triggered this request from the subscriber, taken from the list of events from the subscription request.
`container.id` | Required | string | Resource id of the contextual subject in which this content resides
`content.timestamp` | Required | string | ISO 8601-2 timestamp in UTC provided by the Hub when it performed the content update
`content` | Required | Bundle | A [FHIR Bundle resource](http://www.hl7.org/fhir/bundle.html) containing the requested content changes.

###### Update Content Notification Event Example
```
{
  "timestamp": "2018-01-08T01:37:05.14",
  "id": "q9v3jubddqt63n1",
  "event": {
    "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
    "hub.event": "update-content",
    "container.id": "43c6d969-23d3-4c9b-a5d2-0a1e0960ead5",
    "content.timestamp": "2019-08-23T21:05:22.59",
    "content": [
      {
        "resourceType": "Bundle",
        "id": "8i7tbu6fby5ftfbku6fniuf",
        "type": "collection",
        "entry": [
          {
            "request": {
              "method": "PUT"
            },
            "resource": {
              "resourceType": "Observation",
              "id": "bmd",
              "meta": {
                "source": "app_example_client_id"
              },
              "text": {
                "status": "generated"
              },
              "partOf": [
                {
                  "reference": "ImagingStudy/f65fd7e0-14d5-4202-aa17-249957fe2803"
                }
              ],
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
                "reference": "Patient/160f4ff2-6a32-45e4-810d-829d7d578442"
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
          },
          {
            "request": {
              "method": "POST"
            },
            "resource": {
              "resourceType": "Observation",
              "id": "head-circumference",
              "meta": {
                "source": "app_example_client_id",
                "tag": [
                  {
                    "coding": [
                      {
                        "system": "http://fhircast.org/exchange/protection",
                        "code": "restrict.update"
                      }
                    ]
                  }
                ]
              },
              "partOf": [
                {
                  "reference": "ImagingStudy/f65fd7e0-14d5-4202-aa17-249957fe2803"
                }
              ],
              "status": "preliminary",
              "code": {
                "coding": [
                  {
                    "system": "http://loinc.org",
                    "code": "9843-4",
                    "display": "Head Occipital-frontal circumference"
                  }
                ],
                "text": "Head Circumference"
              },
              "subject": {
                "reference": "Patient/160f4ff2-6a32-45e4-810d-829d7d578442"
              },
              "valueQuantity": {
                "value": 51.2,
                "unit": "cm",
                "system": "http://unitsofmeasure.org",
                "code": "cm"
              }
            }
          }
        ]
      }
    ]
  }
}
```

* `entry.request.method` type of operation the user performed on the resource
    * POST = User created the Resource
    * DELETE = User deleted the Resource
    * PUT = User updated the Resource
* `entry.request.resource` the *complete* resource object for PUT and POST operations or the resource object including the resource id for DELETE operations

### Get Current Content Request
A subscribing application may request the current content of a context container with an HTTP GET to the `hub.topic` url.  The client
queries the case by the Hub's `hub.topic` url with the `container.id` as a parameter.  Note that no `hub.event` is present in the
response.  If the query is successful the Hub shall respond by returning a success HTTP status (200 - OK), HTTP status (204 - No Content),
or returning an [HTTP error status](#get-current-content-request-failure) if the request was not successful.  The subscriber SHALL be able to appropriately handle
a failed request.

##### Response Get Current Content Parameters
Upon successful completion of the request the Hub SHALL return the following response in addition to returning the HTTP 200 OK status.

Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | Event identifier which was uniquely generated by the requesting subscriber.
`hub.event` | Required | string | `get-current-content`<br><br>The event that triggered this request from the subscriber.
`container.id` | Required | string | Resource id of the contextual subject in which this content resides
`content.timestamp` | Required | string | Timestamp provided by the Hub when it last updated content
`content` | Required | array | Resources which are present in the containing contextual subject

###### Get Current Content Example Request

```
GET https://hub.example.com/7jaa86kgdudewiaq0wtu?container.id=43c6d969-23d3-4c9b-a5d2-0a1e0960ead5 HTTP/1.1
Host: hub
Authorization: Bearer i8hweunweunweofiwweoijewiwe
Content-Type: application/json
```
###### Get Current Content Example Response
```
{
  "hub.topic": "https://hub.example.com/7jaa86kgdudewiaq0wtu",
  "hub.event": "get-current-content",
  "container.id": "43c6d969-23d3-4c9b-a5d2-0a1e0960ead5",
  "content.timestamp": "2019-08-23T21:05:22.59",
  "content": [
    {
      "resourceType": "Bundle",
      "id": "8i7tbu6fby5ftfbku6fniuf",
      "type": "collection",
      "entry": [
        {
          "request": {
            "method": "PUT"
          },
          "resource": {
            "resourceType": "Observation",
            "id": "bmd",
            "meta": {
              "source": "app_example_client_id"
            },
            "text": {
              "status": "generated"
            },
            "partOf": [
              {
                "reference": "ImagingStudy/f65fd7e0-14d5-4202-aa17-249957fe2803"
              }
            ],
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
              "reference": "Patient/160f4ff2-6a32-45e4-810d-829d7d578442"
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
        },
        {
          "request": {
            "method": "POST"
          },
          "resource": {
            "resourceType": "Observation",
            "id": "head-circumference",
            "meta": {
              "source": "app_example_client_id",
              "tag": [
                {
                  "coding": [
                    {
                      "system": "http://fhircast.org/exchange/protection",
                      "code": "restrict.update"
                    }
                  ]
                }
              ]
            },
            "partOf": [
              {
                "reference": "ImagingStudy/f65fd7e0-14d5-4202-aa17-249957fe2803"
              }
            ],
            "status": "preliminary",
            "code": {
              "coding": [
                {
                  "system": "http://loinc.org",
                  "code": "9843-4",
                  "display": "Head Occipital-frontal circumference"
                }
              ],
              "text": "Head Circumference"
            },
            "subject": {
              "reference": "Patient/160f4ff2-6a32-45e4-810d-829d7d578442"
            },
            "valueQuantity": {
              "value": 51.2,
              "unit": "cm",
              "system": "http://unitsofmeasure.org",
              "code": "cm"
            }
          }
        }
      ]
    }
  ]
}
```

The `content` field holds the resources which are content of the containing contextual subject `container.id`.

##### Get Current Content Request Failure
If the get current content request fails, the Hub SHALL respond with one of the below HTTP error status codes corresponding with the
described failure condition.

###### Get Current Content Failure - HTTP Error Status Codes
Code | Meaning | Condition
--- | --- | ---
400 | Bad Request | Some portion of the request was improperly formed
401 | Unauthorized | The application was not granted an authorization scope required to get the case's current context
410 | Gone | The containing context resource id provided in the request was not found
412 | Precondition Failed | The application is not subscribed to the `hub.topic`
5xx | *per code* | Appropriate status code if a server error occurs

###### Request Get Current Content Failure - Response
If the get current content request fails, the Hub MAY respond with a [FHIR Operation Outcome resource](http://www.hl7.org/fhir/operationoutcome.html)
in addition to returning the appropriate HTTP error status code.  The operation outcome `severity` SHALL be 'error' while
the following `code` and `details` values are recommended:

HTTP Status | Code | Details
--- | --- | ---
400 | structure | MSG_BAD_SYNTAX
401 | forbidden | MSG_AUTH_REQUIRED
410 | not-found | MSG_NO_EXIST
412 | unknown | MSG_AUTH_REQUIRED
5xx | transient | *not applicable*

### Set Active Resource Request
A subscriber may request a resource be noted as being active with an HTTP POST to the `hub.topic` url.  The Hub SHALL either acknowledge
that the request was honored successfully by returning a success HTTP status (200 - OK) or returning
an [HTTP error status](#set-active-resource-request-failure) if the request was not successful. Support of this operation by a Hub is
optional even for Hub's wishing to support the sharing of structured information.  If a Hub does not support the set active resource
operation then it would return the standard not implemented HTTP error status (501 - Not Implemented) if the operation is requested. The
subscriber SHALL be able to appropriately handle a failed set active resource request.

Noting a resource as active allows applications to react to a user’s selection of structured information in another application.
Hence an application may want to set (or clear) an active resource with others reacting to the selection if appropriate.  For example,
a reporting system may indicate that the user has selected a particular observation with a measurement value and an imaging application
which created the measurement may wish to change its user display such that the image from which the measurement was acquired is visible.
If a resource is noted as active, this indicates that any other resource which was active is no longer active (i.e. there is an implicit
inactive).

###### Set Active Resource Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | *string* | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy. 
`id` | Required | *string* | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID.
`event` | Required | *object* | A json object describing the event. See [below](#set-active-resource-in-case-event-object-parameters).

###### Set Active Resource Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event` | Required | string | `remove-resource-from-case`<br>The event that triggered this request from the subscriber, taken from the list of events from the subscription request.
`container.id` | Required | string | Resource id of the contextual subject in which this content resides
`content.timestamp` | Required | string | Timestamp provided by the Hub when it last updated content
`resource.id` | Required | string | FHIR resource id that should be noted as active, if `resource.id` is an empty string then the requesting application is noting that nothing is selected

##### Response Set Active Resource Parameters
Upon successful completion of the request the Hub SHALL return the following response in addition to returning the HTTP 200 status.

Field | Optionality | Type | Description
--- | --- | --- | ---
`id` | Required | string | Event identifier which was uniquely generated by the requesting subscriber.
`hub.event` | Required | string | `set-active-resource-in-case`<br><br>The event that triggered this request from the subscriber.
`content.timestamp` | Required | string | Timestamp generated by the Hub when it changed the selected resource based on this request.  This timestamp is used to ensure subsequent request from any subscriber is using the current version of the content.

##### Set Active Resource Request Failure
If the set active resource request fails, the Hub SHALL respond with one of the below HTTP error status codes corresponding with the
described failure condition.

###### Set Active Failure - HTTP Error Status Codes
Code | Meaning | Condition
--- | --- | ---
400 | Bad Request | Some portion of the request was improperly formed
401 | Unauthorized | The application was not granted an authorization scope required to note as active the resource type of the FHIR resource requested to be made active
404	| Not Found	| The resource which is requested to be noted as active was not found in the case
410 | Gone | The containing context resource id provided in the request was not found
412 | Precondition Failed | The application is not subscribed to the `hub.topic`
428 | Precondition Required | The `content.timestamp` in the submitted request is not the current timestamp of the content held by the Hub
5xx | *per code* | Appropriate status code if a server error occurs

###### Request Set Active Resource Failure - Response
If the set active resource request fails, the Hub MAY respond with a [FHIR Operation Outcome resource](http://www.hl7.org/fhir/operationoutcome.html)
in addition to returning the appropriate HTTP error status code.  The operation outcome `severity` SHALL be 'error' while
the following `code` and `details` values are recommended:

HTTP Status | Code | Details
--- | --- | ---
400 | structure | MSG_BAD_SYNTAX
401 | forbidden | MSG_AUTH_REQUIRED
404 | not-found | MSG_NO_EXIST
410 | not-found | MSG_NO_EXIST
412 | unknown | MSG_AUTH_REQUIRED
428 | conflict | MSG_VERSION_AWARE_CONFLICT
5xx | transient | *not applicable*

##### Set Active Resource Notification Parameters
The Hub SHALL send the following event to all other applications participating in the case if the request was successfully completed.  The requesting application
SHALL not be sent this event.  The Hub SHALL not send any notification to other applications if the request fails.

Field | Optionality | Type | Description
--- | --- | --- | ---
`timestamp` | Required | string | ISO 8601-2 timestamp in UTC describing the time at which the event occurred with subsecond accuracy.
`id` | Required | string | Event identifier used to recognize retried notifications. This id SHALL be uniquely generated by the subscriber and could be a GUID. Following an accepted context change request, the hub MAY re-use this value in the broadcasted event notifications.
`event` | Required | object | A json object describing the event, see [below](#set-active-resource-in-case-notification-event-object-parameters).

##### Set Active Resource Notification Event Object Parameters
Field | Optionality | Type | Description
--- | --- | --- | ---
`hub.topic` | Required | string | The topic session URI given in the subscription request.
`hub.event` | Required | string | `remove-resource-from-case`<br>The event that triggered this request from the subscriber, taken from the list of events from the subscription request.
`container.id` | Required | string | Resource id of the contextual subject in which this content resides
`content.timestamp` | Required | string | Timestamp provided by the Hub when it last updated content
`resource.id` | Required | string | FHIR resource id that should be noted as active, if `resource.id` is an empty string then no resource is active

### Hub Responsibilities to Support Sharing of Structured Information
In order to support the sharing of structured information a Hub must support the following responsibilities:
  1. Support the [update content](#update-content-request) and [get current content](#get-current-content-request) requests
  2. Support of the [set active resource](#set-active-resource-request) is optional
  3. Maintain the current state of shared resources in order to support the [get current content](#get-current-content-request) and validate parameters of requested operations as described in each operation
  4. Optionally enforce data protection levels
  5. Ensure the structural validity of FHIR resources
  6. Ensure that a specific `client.id` in any request uses the same token as when a given `client.id` was first encountered

A Hub is not responsible for any long-term persistence of shared information and should purge the content when the containing contextual
subject is closed.

