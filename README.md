# customer-api-raml

Sample API specification developed in RAML for managing customers.

The API specification is designed to meet the following use cases:

1. Allowing automated API consumers (for example, cron jobs or middleware applications) invoke APIs to fetch list of customers.
2. An end-user such as customer service representative should be able to use a mobile device to manage customers using the APIs
3. In current state, the API specification only deals with customers. We should be able to expand to other resources such as orders and products with relative ease.

## 1. Demo

To preview the API specification in your browser, please see [here for raml2html output](https://rawgit.com/aimtiaz11/customer-api-raml/master/output/index.html).

## 2. Assumptions

In this exercise, we will make the assumption that the API server is protected by a Authorization Server which will do the OAuth verifications.

## 3. Security considerations

### 3.1 HTTPS

To encrypt traffic between the API consumer and API server, HTTPS will be used in conjunction with TLS. This will prevent man-in-the-middle attacks, packet sniffing and other similar malicious attacks.

Using HTTPS with TLS 1.2 is important as the APIs are internet facing (as opposed to point-to-point in a data centre or within a WAN/LAN setup). This warrants all communications to be encrypted end-to-end. TLS 1.0 is currently more widely adopted, however major API vendors are [slowly deprecating it in favour of TLS 1.2](https://www.thesslstore.com/blog/deprecation-tls-1-0-1-1-underway/).


### 3.2 OAuth for Authorization

The APIs will be protected using OAuth 2.0. OAuth 2.0 is deemed more secure than other methods such as API keys and Basic Auth as user authentication credentials are not passed around in every request. Although, a downside is that implementing OAuth is a bit more cumbersome than simple API keys or Basic Auth.

We will support two variants of OAuth - two-legged (application authentication) and three-legged (user authentication) and both are intended to satisfy the use cases 1 and 2 above.

#### 3.2.1 Application Authentication

Application authentication (or two-legged OAuth) does not require any user context. The Authorization process involves:

1. An API client issued with `client_id` and `client_secret`
2. An API Authorization server validating the request, issuing the access and refresh tokens (discussed below in more details)

API client will need to keep the `client_id` and `client_secret` encrypted for security.

This is suitable for applications or automated scripts that run at regular interval to fetch client data.

#### 3.2.2 User Authentication

This is the traditional OAuth scenario (three-legged scenario) where the API Client is redirect to the Authorization server to enter their credentials. Upon successful authentication, the client receives the access and refresh tokens that will be used to communicate with the API server.

This is suitable for mobile or web application clients with an end-user involved.


#### 3.2.3 The difference between User and Application Authentication

Clients who obtain their token via _Application Authentication_ will only have read-only access to the data. They can only invoke the safe HTTP methods such as GET which does not manipulate any resource.

## 4. Consumer Integration

In this section we describe how we can fulfil first two use cases.

### 4.1 Automated access to the API (use case 1)

An API consumer that wish to invoke customer APIs periodically will need to register with the API service in order to obtain client ID and client secret.


Once they have obtained that, the rest of the OAuth Authorization will follow as below:

**Step 1:** API consumer sends a request for token by sending the `client_id` and `client_secret` obtained during registration in an `Authorization` header as Base64 encoded string.

```
POST /oauth/token HTTP/1.1
Host: api.samplehost.com
Authorization: Basic Base64(client_id:client_secret)

```

**Step 2:** The Authorization server will respond with the token:

```json
{
    "token_type": "bearer",
    "access_token": "79c11a62df3d4d92c092314071941e4489c8f8e7",
    "refresh_token": "0b12356dfbd0980ed3347f4acf6a824d06b7b8e3"
}
```

**Step 3:** The client can then use the `access_token` for all subsequent API calls by specifying it in the `Authorization` header.

```http
GET /api/v1/customers?pageNo=0 HTTP/1.1
Host: api.samplehost.com
Authorization: Bearer 79c11a62df3d4d92c092314071941e4489c8f8e7
```

> The `refresh_token` can be used to obtain a new `access_token` when that expires.


#### 4.1.1 Request pagination

Due to the high volume of data that can be retrieved by the API which can affect performance, we need to enforce pagination which will restrict the amount of data that can be fetched per API call.

This will help reduce load on the server as the API's consumer base scales up.

This means that the API client will need to read the pagination attributes such as `pageSize`, `totalPages`, `first` and `last` to determine how many subsequent requests to make to extract the complete collection of the customer resource.

#### 4.1.2 Conditional Requests

Conditional requests allow API client to validate whether their cached copy of the customer resource or collection is valid or not. It makes use of the `ETag`/`If-None-Match` headers for single resource and `Last-Modified`/`If-Modified-Since` for collections.

### 4.1.2.2 Conditional Request for resource

API client will need to cache or store the `ETag` header value of the HTTP 200 OK Response and pass that to subsequent `GET /api/v1/customer/{customerId}` requests using a `If-None-Match` header.

If the server determines that the resource has been modified based on the `If-None-Match` header value, then will respond with a HTTP 200 OK with the body containing the new Customer resource.

If the resource has not been modified, the API server will respond with a HTTP 304 Not Modified with empty body.

### 4.1.2.2 Conditional Request for collection

When automated API consumers will try to consume the API, they will need to perform an initial load
of the full customer dataset. This will need to happen by iterating through all the pages of customer result set in one iteration.

In second iteration, the API consumer will only want records that have been updated. So after the initial data load, the API consumer will need to pass the `If-Modified-Since` with the value since the last time the API call was made. This will return those customer resources that have been updated since the last API call.




Both the pagination feature and conditional request feature will help prevent server and network from being overloaded by expensive API calls and allow the automated API client to make fewer API calls to fetch full result set.


### 4.2 Mobile and IoT integration (use case 2)

#### 4.2.1 Authentication Process

Mobile and IoT devices should leverage the the `User Authentication` (three-legged OAuth) which will give them access to full range of APIs that the API service has to offer.

All mobile applications will need a `client_id` and `client_secret` which they will obtain after registering their mobile device or application with the API service.

End-users such as a Customer Service Representative using the application, will need to authenticate in the Authorization server which will issue the the end-user's device with the access token.

The authentication flow will be as follows:

1. A customer service agent attempts to login in the mobile application.
2. The mobile application redirects end-user to the Authorization Server which will present a page/screen for customer service agent to enter their username and password.
3. Customer service agent will enter their own login credentials and upon submission, the Authorization Server will redirect back to the client application URL (Authorization grant process)
4. The mobile application will extract the `authorization_code` from the response and call `/oauth2/authorize` to obtain the `access_token` and `refresh_token`
5. The Authorization server will return the `access_token` and `refresh_token` back to the mobile application.
6. The mobile application will use `access_token` for all future interactions with the API Server.


#### 4.2.2 Developing around the APIs

The API uses standard response model which look like below:
```
{
  "success": true,
  "status": 200,
  "data": {
    // actual response data goes here
  }
}
```
We use this object for all HTTP responses - whether its 200 OK or 400 Bad Request or any other response types.

This way, application developers can easily write cleaner client-side code as they can always expect a single standard 3 attribute response object as response.


## 5.1 Adding other resources (use case 3)

The API spec has been developed to make it easier further expand it to include other resources such as `orders` and `products`.

### 5.1.2 Heavy use resource types and parameterisation

In the API spec, we defined two resource type called *collectionType* and *resourceType*.

*collectionType* is designed to be used on a collection (of resources). For example, `/customers` or `/accounts` or `/products`.

*resourceType* is applicable to individual resource - a single item in the collection.

For example, `/customers/{customerId}`, `orders/{orderId}`.

The applicable HTTP methods (GET, PATCH, PUT, POST, DELETE) has been applied directly to the resource types instead of the resource themselves.

`collectionType` allows the following method:
1. GET - Retrieve the whole collection of resources
2. POST - Create a new resource in the collection

`resourceType` allows the following methods:
1. GET - Retrieve resource
2. PUT - Update resource
3. DELETE - Delete resource

This way, we can simply add new resources and associate them with the resource types and it will cover the common API methods automatically.

Note: There may be cases were we may not want to allow a certain API operation on a collection or resource. For example, deleting customers might not be desirable, in this cases we would remove the HTTP DELETE method away from the resource type and apply it on the API definitions of the resource themselves.

In real life, we might not put a destructive operation like DELETE under resource type but rather directly on the collection or resources in the API definitions.


#### 5.1.2.1 Example of how to expand the API

Suppose we are adding the following new resources: `orders` and `products`. An use case around these new resources is that `customer` places `orders` and `orders` will contain number of `products`. So there is a one-to-many between `customers` and one-to-many between `orders` and `products`.

Using these relationships, the API URLs might look like below:

1. `/customers/{customerId}/orders`: All orders that belong to a customer
2. `/customers/{customerId}/orders/{orderId}`: Specific order that belong to a customer
3. `/customers/{customerId}/orders/{orderId}/products`: List of products in an order that customer placed
4. `/products`: A collection of all products
5. `/products/{productId}`: A particular product

The following snippet shows how we could introduce a new `order` resource and nest it within the current `customer` resource:

```raml

### Types
types:
  customer:
    description: Model for 'Customer' resource
    schema: !include schemas/customer.schema
  order:
    description: Model for 'order' resource
    schema: !include schema/order.schema

### API definition
/customers:
 ### content removed for brevity ###
  /orders:
    type: {
        ## Associate type 'collectionType' to orders
        collectionType:
              {
                examplePaginatedResponseExample : !include examples/getCustomerOrders-example.json,
                examplePaginatedResponseType: paginatedResponse,

                createResourceRequestExample : !include examples/order-example.json,
                createResourceRequestType: order,

                createResourceResponseExample: !include examples/createOrderResponse-example.json,
                createResourceResponseType: order,

                createResourceBadRequestExample: !include examples/badRequestExample.json
              }
        }
  /orders/{orderId}:
    ## Associate type 'resourceType' with individual order item
    type: {
        resourceType:
          {
            updateResourceRequestExample : !include examples/order-example.json,
            updateResourceRequestType: order,

            updateResourceResponseExample: !include examples/order-example.json,
            updateResourceResponseType: order,

            getSingleResourceExample: !include examples/order-example.json,
            getSingleResourceType: order
          }
        }
```

> Note: The JSON schemas and samples responses need to be created as first step before the above API setups can be added.


Therefore, by associating the API methods with the resource types, we make this easily scalable to other resources.


# 6. Other design decisions

## 6.1 Excluding hypermedia links (HATEOAS)

Strict adherence to REST standards mandates having hypermedia links and in the community there are strong proponents both for and against having hypermedia links in the response body.

I have deliberately left out hypermedia links, as they do not really serve any purpose as far as the use cases are concerned. On the other hand, they make API response more bloated.
