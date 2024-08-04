# RFC for API standard STNT

## Overview

In order to make APIs between front-end and services easier to use,
this is a proposal for an API standard. It defines rmeote APIs look and feel. 
In other words, this is a style guide for APIs. 

### What is not covered

This proposal is also only intended for request-response traffic, and does not apply to any event based (pub/sub) communication.

## Proposal

### Method Names

All remote APIs (hereby just called APIs) should be via request-response and a globally unique method name.
Handler methods must be globally unique, and will never be "namespaced".

If using multiple services, a discovery service or message bus can be used to ensure the correct messages are routed to the correct service.
Services names themselves should not be in routes.

Namespacing on services or domains is not used because it tightly couples a specific endpoint to a specific service. 
It is common for larger services to be broken up over time as business context evolves. 
With tight coupling you would not be able to move the endpoints to other service 
(aka, break up the service into more pieces) without also changing every client
to call the new url. For example, if you had `foobar-service` and was breaking it up into `foo-service` and `bar-service`,
you would need `/foobar/bars` to become `/bars/bars` which is not feasible to change everywhere once you have a larger number of clients.
To prevent the coupling, all URLs are global, and not namespaced. This means that creation of new endpoints requires research to make sure that
no other service has similar named methods already.

Methods will always be in the format of `{noun}.{verb}` and not include anything else. The noun is the object
that the method is acting on, and the verb is the action. The noun should always be plural, and the method itself
should always be batch. _There will be no singular item methods_. A method will always be default batch.

#### Example

An example of such a method is `authors.list`, which would list authors. If the client wanted a single author,
they could add the filter for `author_ids: [1234]` and get only the `1234` author included in the response.

#### Allowed Methods

The following method verbs should be standard names for any object. All methods are optional, and it is never required to implement all of them:

1. list
2. update
3. create
4. delete
5. duplicate

For example, `authors.list`, `authors.update`, `authors.create`, `authors.delete`.  
These four verbs should always be used instead of alternatives like "add", "edit", "modify", "get".
Updates are always a "patch" and never a "put". That is to say, they are merged with the remote resource, and never completely replace 

Special verbs. The following verbs can also be used to modify a relationship or subitem on a resource.
They should not always be used, but can be used when the relationship between the resource and subitem is
complex, or is not easily modeled as a standard attribute of the main resource.

1. add-{subitem}
2. modify-{subitem}
3. remove-{subitem}

For example, a library and a book have a relationship, but you dont typically modify books on the library
when you modify the library, because adding a book is more like adding inventory.
So instead of modifying books in a complex JSON object in libraries, we can
add these special methods.

As an example:

- `libraries.add-books`: Adds existing books to a library.
- `libraries.modify-books`: Modifies the books in a library.
- `libraries.remove-books`: Removes existing bookes from a library.

Other verbs or methods should be allowed on a resource as needed.
Such verbs must follow `kebab-casing` and have a dash (`-`) to separate words.

Resources should be a single word if possible, but otherwise noun-words should be separated by an underscore (`_`).

An example of these two together: `recording_artists.add-voice_types`. You will notice the mix of `-` and `_`.
The `-` separates verbs from nouns as well as verb works, and the `_` separates noun words.

### Body Syntax

1. The response body of a method should always include a top level object which is the name of the resource being listed or modified.

   The exception is if there is no response at all (as will be common on non list methods)
   As an example, if you are listing authors, then the JSON body would look like:

   ```json
   {
   "authors":
     [{...},
      {...}
     ]
   }
   ```

   Where `{...}` represents a single record.

   Even if only one object was requested, the result should be a list of those objects.

2. All attributes or a resource must be in the format `snake_case`.
   No upper case letters, each word separated by a `_`.

3. All methods that cause side-effects, do not return the objects after modifying them, but instead return a list of events which represent the side-effects.
   This would include all `delete`, `update`, etc. Ths would mean the body for updating an author would look like:
   
   ```json
   {
    "_events":
     [{...},
      {...}
    ]
   }
   ```

5. Meta fields all begin with an underscore (`_`) to mark that they are not part of the resource itself. The following meta fields should be used where applicable.

   - `_includes`: Included resources that are not the resource itself. These can typically be requested on a `list` call, for example, you can have
     `authors.list` and the attribute `_include: ['books']` for the response to include all related books as well as the authors.
     `_includes` should be a flattened list. For example, if you get multiple authors, you get all books across all the authors.
     They should not be nested "per author"
   - `_events`: These are events that represent side-effects. 
     `_events` will have a list of events that were part of the synchronous handling of the primary action.
   - `_meta`: Any other data about the response that isn't related to the resource itself, such as pagination information.

All meta fields are optional, but any data about a request that isn't the resource itself needs to be in a meta field.

### HTTP Specific Rules

This specification is transport agnostic, so anything unique to HTTP isn't used in order to make using other
transports future compatible. For this purpose, if using HTTP as the method of transport, the following rules apply.

1. The request method should _always_ be a `POST`.
2. There should be no parameters in the path or query string. Everything goes in the body.
   This is one reason why `POST` is always used.

### Headers

For information that is needed on a request, but does not make sense as a part of the request body, headers can be used.
For example the following headers are common.

- `X-CORRELATION-ID` : This header should be passed throughout the entire system. Makes it easier to trace distributed events.
  This header should be added by the web-gateway or initial service (not required by clients). Services should always log it in errors as well as propagate it through all remote calls.
- `X-CSRF-TOKEN` : The csrf (or xsrf) token to prevent CSRF attacks. This is needed on every request if your service needs to handle CSRF (cross-origin).

### Errors

Errors should return a status code which matches HTTP status codes where applicable.
Even if not using HTTP, the status code can make programming a specific class of errors easier.
If not using HTTP, the status code should be the `status_code` attribute of the error returned.

Errors will be first class objects of the response, meaning that the body of an error looks like:

```json
{
  "error": {
     "message": "some error",
     "description": "[Extra Code] This is a longer description about what happened. \nMultiple lines is possible. This will only respond when in debug mode or on localhost. This will also only show up if the service has implemented it."
      ...
  }
}
```

The error may contain attributes unique to itself based on the status code. All errors must contain a `"message"` attribute.

### Status Codes

The following status codes should be used where applicable:

- `200`: Everything is ok. The request worked as expected.
- `201`: Created. This should be used on a `create` when everything worked correctly.
- `400`: The request is invalid. This is a programming error on behalf of the client or some value doesn't pass validation.
  For example, it could be from a missing parameter, or because the user typed in an invalid formula.
- `401`: Not Authenticated. No credentials. This should be used when a request doesn't include any credentials.
- `403`: Forbidden. Used when the user credentials are valid, but they lack the appropriate authorization/permissions for this specific request.
- `404`: Not found. Used when a requested resource doesn't exist. This should not be a common status code, because all methods are batch, which means the expected
  response would be a 200 with an empty list instead of a 404.
- `500`: Server error. This essentially denotes a backend bug, and should alert the team to fix something. 500 is the correct status code
  when the server would break, due to programming errors.
- `503`: Service unavailable. This status is used when a down-stream network dependency isn't available.
  It essentially means that there isn't an error, just something is down. 503's should be able to be resolved by "trying again later".
- `502`/`504`: will only be used by infrastructure peices and should be thought of similar to `503` but should never be directly returned.

### Pagination

Pagination will be done via the `_meta` attribute in a response. Not all APIs are required to use pagination.
Therefore the `_meta` field and the `pagination` block are both optional. Clients should be aware that they might not exist.

If you are paginating, the initial request doesn't need any special parameter. The response should include the following in the body:

```json
{
  "_pagination": {
    "offset": 200,
    "limit": 100,
    "total": 432,
  }
}
```

The client know the `offset` and the `limit` from the response. The client needs to use this to infer the next and previous pages when using `offset` in the request

A client would then use the following in the request:

```json
{
"_pagination": {
  "offset": 200,
  "limit": 100  // optional
}
```

## Versioning

The specification purposely does not cover versioning. 

## Examples

### list example

Request:

`POST authors.list`

body:

```json
{
  "author_id": "a.esg62odfcfa3z",
  "_include": ["books"]
}
```

Result:

headers:

```
status: 200
```

body:

```json
{
  "authors": [
    {
      "id": "a.6y02ql2ypuy711",
      "name": "Sam Newman"
    },
    {
      "id": "a.q7ip0q40hcdb5c",
      "name": "Donald E. Knuth"
    }
  ],
  "_includes": {
    "books": [
      {
        "id": "b.yq27az1y",
        "name": "The Art of programming",
        "author_id": "a.q7ip0q40hcdb5c"
      },
      {
        "id": "b.yipbbbma",
        "name": "Building Microservices",
        "author_id": "a.6y02ql2ypuy711"
      }
    ]
  }
}
```

### Update Example

Request:

`POST authors.update`

body:

```json
{
  "authors": [
    {
      "id": "a.6y02ql2ypuy711",
      "name": "T.R. Ried"
    }
  ],
}
```

Result:

headers:

```
status: 200
```

body:

```json
{
  "_events": [{
    "event_name": "author.updated",
    "payload": {
        "author": [
          {
            "id": "a.6y02ql2ypuy711",
            "updated_ts": "2020-10-21T18:48:12.985Z",
            "name": "T.R. Ried"
          }
        ]
    }
  }],
 }
```

### Create Example

Request:

`POST authors.create`

body:

```json
{
  "authors": [
    {
      "name": "Andrew McAfee"
    }
  ]
}
```

Result:

headers:

```
status: 201
```

body:

```json
{
  "_events": [
    {
     "event_name": "author.created",
     "payload": {
        "id": "a.6y02ql2ypuy711",
        "name": "Andrew McAfee"
    }
  ]
}
```

### Delete Example

Request:

`POST authors.delete`

body:

```json
{
  "ids": ["a.6y02ql2ypuy711"]
}
```

Result:

headers:

```
status: 200
```

body:

```json
{
 "_events": [
  {
   "event_name": "author.deleted",
   "payload": {
     "id": "a.q7ip0q40hcdb5c"
   }
  },
  {
    "event_name": "book.deleted",
    "payload": {
      "id": "b.yq27az1y"
     }
   }
 ]
}
```

## FAQ

- Why no path parameters? The goal is to be transport agnostic. And transports such as ZeroMQ have static routes with
  no construct of dynamic routes. Also, sometimes path parameters get ambiguous. Should you put all the ids in the path? or just one? How nested can things go?
  By removing the path parameters and keeping things flat, you keep things less ambiguous. From a metrics point of view
  it is also hard to track metrics on a specific endpoint if you have dynamic paths. A special tool which is endpoint aware needs to be used
  rather than simple string matching. The complexities of path parameters (as well as query parameters) doesn't justify the pros.
