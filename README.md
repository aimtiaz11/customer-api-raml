# customer-api-raml

API documentation for managing customers written in RAML. The API is designed to satisfy the following requirements:

1. Allowing automated API consumers (cron jobs or middleware applications) invoke APIs to fetch list of customers.
2. Allow API clients like Postman or mobile devices (for example, an app in iOS or Android) invoke APIs.
3. Security - End-to-end encryption
4. Scalability - In current state, the API specification only deals with customers. We should be able to expand to other resources with ease.

## 1. Demo
To preview the API specification in your browser, please see [here](https://rawgit.com/aimtiaz11/customer-api-raml/master/output/index.html).


## 2. Security considerations

### HTTPS

To encrypt traffic between the API consumer and API server, HTTPS will be used in conjunction with TLS. This will prevent man-in-the-middle attacks, packet sniffing and other similar malicious attacks.

Using HTTPS with TLS 1.2 is important as the APIs are internet facing (as opposed to point-to-point in a data centre or WAN/LAN). This warrants all communications to be encrypted end-to-end. TLS 1.0 is currently more widely adopted, however major API vendors are [slowly deprecating it in favour of TLS 1.2](https://www.thesslstore.com/blog/deprecation-tls-1-0-1-1-underway/).


### OAuth

The APIs will be protected using OAuth 2.0. OAuth 2.0 is deemed more secure than other methods such as API keys and Basic Auth as user authentication credentials are not passed around in every request. Although, a downside is that implementing OAuth is a bit more cumbersome than simple API keys or Basic Auth.

We will support both two and three-legged OAuth.


Two-legged OAuth does not require any human context. The Authorization process involves:

1. An API client issued with `client_id` and `client_secret`
2. An API Authorization server validating the request.



## 3. Consumer integration

### 3.1 Automated access to the API
### 3.2 Mobile and IoT integration


### 4. Additional consideration

#### 4.1 Pagination

#### 4.2 Conditional requests

#### 4.3 Rate limiting
