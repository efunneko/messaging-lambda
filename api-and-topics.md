This document contains information on the API between components in messaging-lambda.


The API uses REST-over-message-bus, where the URL is the topic and all messages
contain JSON with the form:

{
  action: "GET|POST|PUT|PATCH|DELETE",
  params: {  // params represent URL parameters
    param-1: "param-1-val"
    ...
  },
  objects: [ 
    { // object 1
      property-1: "property-1-val",
      ...
    }
  ]
}

