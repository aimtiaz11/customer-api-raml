# customer-api-raml

API documentation for managing customers written in RAML. The API is designed to satisfy the following use cases:

1. Allowing automated API consumers (for example, cron jobs or middleware applications) invoke APIs to fetch list of customers.
2. Allow API clients such as mobile apps (ore even web application clients) to interact with the APIs.  
3. In current state, the API specification only deals with customers. We should be able to expand to other resources such as orders and products with relative ease.

## 1. Demo
To preview the API specification in your browser, please see [here](https://rawgit.com/aimtiaz11/customer-api-raml/master/output/index.html).


## 2. Security considerations

### 2.1 HTTPS

To encrypt traffic between the API consumer and API server, HTTPS will be used in conjunction with TLS. This will prevent man-in-the-middle attacks, packet sniffing and other similar malicious attacks.

Using HTTPS with TLS 1.2 is important as the APIs are internet facing (as opposed to point-to-point in a data centre or WAN/LAN). This warrants all communications to be encrypted end-to-end. TLS 1.0 is currently more widely adopted, however major API vendors are [slowly deprecating it in favour of TLS 1.2](https://www.thesslstore.com/blog/deprecation-tls-1-0-1-1-underway/).


### 2.2 OAuth for authorization

The APIs will be protected using OAuth 2.0. OAuth 2.0 is deemed more secure than other methods such as API keys and Basic Auth as user authentication credentials are not passed around in every request. Although, a downside is that implementing OAuth is a bit more cumbersome than simple API keys or Basic Auth.

We will support two variants of OAuth - two-legged (application authentication) and three-legged (user authentication) and both are intended to satisfy the use case 1 and 2.

#### 2.2.1 Application authentication

Application authentication (or two-legged OAuth) does not require any user context. The Authorization process involves:

1. An API client issued with `client_id` and `client_secret`
2. An API Authorization server validating the request.

API client will need to keep the `client_id` and `client_secret` encrypted for security.

This is suitable for applications or automated scripts that run at regular interval to fetch client data.

#### 2.2.2 User Authentication

This is the traditional OAuth scenario where the API Client is redirect to the Authorization server to enter their credentials. Upon successful authentication, the client receives the access and refresh tokens that will be used to communicate with the API server.

This is suitable for mobile or web application clients.


#### 2.2.3 The difference between User and Application Authentication

Application authentication clients only has read-only access to the data. They are unable to create, update or delete a resource.


## 3. Consumer integration

In this section we describe how we can fulfil the above 2 use cases.

### 3.1 Automated access to the API (use case 1)

An API consumer that wish to invoke customer APIs periodically will need to register with the API service with the client ID and client secret.

Auth flow:

API consumer sends a request for token:

```
POST /oauth/token HTTP/1.1
Host: api.samplehost.com
Authorization: Basic Base64(client_id:client_secret)

```

The Authorization server will response with the token:

```json
{
    "token_type": "bearer",
    "access_token": "sdfasdlkfjasdlfjaaf"
}
```

The client can then use the token for all subsequent API calls.

```http
GET /api/v1/customers?pageNo=0 HTTP/1.1
Host: api.samplehost.com
Authorization: Bearer sdfasdlkfjasdlfjaaf
```


#### 3.1.1 Request pagination

Due to the high volume of data that can be retrieved by the API which can affect performance, we need to enforce pagination. This means that the API client will need to read the pagination attributes such as `pageSize`, `totalPages`, `first` and `last` to determine how many subsequent requests to make to extract the complete collection of resource.

#### 3.1.2 Conditional requests

Conditional requests allow API client to validate whether their cached copy of the customer resource is valid or not. It makes use of the `ETag` and `If-None-Match` headers.

API client will need to cache or store the `ETag` header value of the HTTP 200 OK Response and pass that to a subsequent `GET /api/v1/customer/{customerId}` using a `If-None-Match` header.

If the server determines that the resource has been modified based on the `If-None-Match` header value, then will respond with a HTTP 200 OK response.

If the resource has not been modified, the API server will respond with a HTTP 304 Not Modified with empty body.

Both the pagination feature and conditional request feature prevents server and network from being overloaded by expensive API calls.


### 3.2 Mobile and IoT integration (use case 2)

#### 3.2.1 Authentication process

Mobile and IoT devices should leverage the the `User Authentication` (three-legged OAuth) which will give them access to full range of APIs that the API server has to offer.

All mobile applications will need a `client_id` and `client_secret` which they will obtain after registering their mobile device or application with the API service.

End-users such as a Customer Service Representative using the application, will need to authenticate in the application and obtain an `access_token` which can then be used for

The authentication flow will be as follows:

1. A customer service agent attempts to login in the mobile application.
2. The mobile application redirect to the Authorization Server which will present a page/screen for customer service agent to enter their credentials.
3. Customer service agent will enter the credential and upon submission, the Authorization Server will redirect back to the client application URL (Authorization grant process)
4. The mobile application will extract the `authorization_code` from the response and call `/oauth2/authorize` to obtain the `access_token` and `refresh_token`
5. The Authorization server will return the `access_token` and `refresh_token` back to the mobile application.
6. The API will use `access_token` for all future interactions with the API Server.


#### 3.2.2 Developing around the APIs

The API uses standard response model which look like below:
```json
{
  "success": true,
  "status": 200,
  "data": {
    // actual response data goes here
  }
}
```
We use this object for all HTTP responses, whether its 200 OK or 400 Bad Request.

Application developers can easily write cleaner client side code to handle the response object and extract success response or error messages from the code.


## 4.1 Adding other resources (use case 3)

The API spec has been developed to make it easier further expand it to include other resources such as `orders` and `products`.

### 4.1.2 Heavy use resourceTypes and parameterisation

We defined two resourceType called `collection` and `resource`. `collection` is designed to be used on a collection of resources. For example, `/customers` or `/accounts` or `products`.

`resource` is applicable to individual resource. For example, `/customers/{customerId}`, `orders/{orderId}`.

The applicable HTTP methods (GET, PUT, POST etc) has been applied directly to the resource types instead of the resource themselves.

`collection` allows the following method:
1. GET - Retrieve the whole collection of resources
2. POST - Create a new resource in the collectoin

`resource` allows the following methods:
1. GET - Retrieve resource
2. PUT - Update resource
3. DELETE - Delete resource

This way, we can simply add our new resources and associate them with the resourceType and it will cover the common API methods automatically.


Suppose we are adding the following new resources: `orders` and `products`. An use case around these new resources is that `customer` places `orders` and `orders` have `products`. So there is a one-to-many between `customers` and one-to-many between `orders` and `products`.

The following are some method examples that that we might wish to implement.

1. `/customers/{customerId}/orders`: All orders that belong to a customer
2. `/customers/{customerId}/orders/{orderId}`: Specific order that belong to a customer
3. `/customers/{customerId}/orders/{orderId}/products`: List of products in an order that customer placed
4. `/products`: A collection of all products
5. `/products/{productId}`: Retrieve a product

The following snippet shows how we could add `orders` within `customer`:

```raml
/customers:
 ### content removed for brevity ###
  /orders:
    type: {
        ## Associate type 'collection'
        collection:
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
    ## Associate type 'resource' with individual order item
    type: {
        resource:
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
Therefore by associating the API methods with the resource types, we make this easily scalable to other resources.
