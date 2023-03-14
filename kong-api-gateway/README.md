## Kong Api Gateway setup using Docker
### Installing Kong Api gateway using docker compose
- upstream server setup:
docker run -d -p 6969:80 ealen/echo-server

Reference: https://hub.docker.com/r/ealen/echo-server
##################################################################################################################
we need to files to install kong api gateway using docker compose
	1.docker-compose.yml
	2.config/kong.yml
	
we weill improve these two files setp by step to experiment all feutures

1. Installation
- docker-compose.yml
```
version: '3.3'

services:
  kong:
    image: kong
    volumes:
      - "./config:/usr/local/kong/declarative"
      - "./logs/file.log:/file.log"
    environment:
      - KONG_DATABASE=off
      - KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative/kong.yml
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl
      - KONG_LOG_LEVEL=debug
      - KONG_PLUGINS=bundled
    ports:
      - "8000:8000/tcp"
      - "127.0.0.1:7990:8001/tcp"
      - "8001:8001/tcp"
      - "8443:8443/tcp"
      - "127.0.0.1:8444:8444/tcp"
```
- config/kong.yml
```
_format_version: "2.1"

services:

- name: echo-server
  url: http://100.64.188.202:6969
  routes:
  - name: echo
    paths:
    - /params
```

Now, run docker compose up & hen check on localhost:8000/path	for testing
##################################################################################################################
### Testing cors using kong API

- improve congig/kong.yml by adding plugins section with cors configuration
```
_format_version: "2.1"

services:

- name: echo-server
  url: http://100.64.188.202:6969
  routes:
  - name: echo
    paths:
    - /params
plugins:
- name: cors
  config:
    origins:
    # - '*'
    - https://google.com    
	- http://info.cern.ch
    methods:
    - GET
    - POST
    headers:
    - Accept
    - Accept-Version
    - Content-Length
    - Content-MD5
    - Content-Type
    - Date
    - X-Auth-Token
    - Authorization
    exposed_headers:
    - X-Auth-Token
    credentials: true
    max_age: 3600
    preflight_continue: false
```
- now run docker-compose up and test by following below steps

understand the concept of cors(cross origin resource sharing):
CORS stands for Cross-Origin Resource Sharing, which is a security feature that restricts web pages or APIs from making requests to a different domain than the one the origin came from.
In the context of an API Gateway, CORS is a mechanism that allows a client application running in a browser to access resources on a server that is in a different domain or origin.
API Gateway typically handles CORS by adding special HTTP headers to responses. These headers include "Access-Control-Allow-Origin," "Access-Control-Allow-Methods," and "Access-Control-Allow-Headers," which allow the client application to access the resources it needs.
By configuring CORS in API Gateway, you can ensure that your API is accessible to authorized users and applications, while at the same time preventing unauthorized access and protecting your data from malicious attacks.

In cors configuration the client request protocol must match with server respponse protocol,i.e., http -> http or https -> https
In above code '*' means allwing requests from all origins(domains),allowing methods GET And POST also allowing given headers.

#### Testing
- disable Block insecure private network requests. in chrome://flags/ in your browser to pass https and http conflicts.
- open any http page on browser(ex:http://info.cern.ch/)
- open developer options(inspect),go to network . in console give command
```
fetch("http://localhost:8000/params")
```
- if the cors is allowed from http://info.cern.ch/ url,then you can see no cors blocking,if it is now allowed cors will block the request.as seena attached screenshot

Reference: https://docs.konghq.com/hub/kong-inc/cors/

##################################################################################################################

## Rate Limiting In Kong API
 We have to setup ratelimit plugin in kong api configuration.
understand the concepts of Ratelimiting(uses):
Rate limiting in API refers to the practice of controlling the amount of traffic that is sent to or from an API by setting a limit on the number of requests that can be made in a given period of time. It is a technique used to prevent abusive or excessive use of an API, which can cause service disruptions, degrade performance, or lead to security vulnerabilities.
The rate limit can be set based on a number of factors, such as the number of requests per second, per minute, or per day, the user's identity or API key, or the type of endpoint being accessed. When the rate limit is exceeded, the API will return an error response or delay the request until the next time window.
Rate limiting helps to ensure that API services remain available and performant for all users, while also preventing abuse and unauthorized access. It is a common technique used by many APIs, including popular platforms like Twitter, Google, and Amazon.
 
- changes in config/kong.yml
```
_format_version: "2.1"

services:

- name: echo-server
  url: http://100.64.188.202:6969
  routes:
  - name: echo
    paths:
    - /params
plugins:
- name: cors
  config:
    origins:
    # - '*'
    - https://google.com    
	- http://info.cern.ch
    methods:
    - GET
    - POST
    headers:
    - Accept
    - Accept-Version
    - Content-Length
    - Content-MD5
    - Content-Type
    - Date
    - X-Auth-Token
    - Authorization
    exposed_headers:
    - X-Auth-Token
    credentials: true
    max_age: 3600
    preflight_continue: false
	
- name: rate-limiting
  config: 
#    second: 5 #per second 5 requests
    hour: 10 #per hour 10 requests
    policy: local
    fault_tolerant: true
    hide_client_headers: false
    redis_ssl: false
    redis_ssl_verify: false
# comment these if you don't want to store rate limiting data in redis
#    policy: redis
#    redis_host: 172.27.59.36
#    redis_password: example
```
- you can use three polices for configuring ratelimiting,
    1.local
	2.cluster(not free)
	3.redis
- in the above code we try local and redis. where thhe number will be stored that no.of  times we hit the api.in above code if we hit 10 times  then it will block us.
- the no.of time hit will stored in either local or redis based on our configuration
- if you hit more than 10 time in a hour you will get 429 (too many requests errors) 

Reference: https://docs.konghq.com/hub/kong-inc/rate-limiting/
##################################################################
### Ip Restrictons

Restricting some ips from accessing the API Gateway .or allowing specific Ip,range of Ips or subnets etcs.
- change in config/konf.yml
```
plugins:
- name: ip-restriction
  config: 
    # deny:
    # - 172.28.64.0/24
    allow:
    - 124.7.61.72
    - 192.168.98.176
    - 192.168.98.16
    status: 401
    message: cannot grant access
```
reference: https://docs.konghq.com/hub/kong-inc/ip-restriction/
####################################################################
Bot detection
 - uses regular expressions in config file 
 - you can check for bots regular expression in google/github to deny them access
 - this detects the bots by using a header in rquest called user-agent
 - we can bypass the bot detection by using manual user-agent in curl request,also you  can use extensions of browsers(ex:requestly)
 - changes in config/kong.yml
 ```
 plugins:
 - name: bot-detection
  config:
    deny:
    - "(C|c)hrome"
    - "curl"
  enabled: true
```
Reference: https://docs.konghq.com/hub/kong-inc/bot-detection/
#############################################################
### Proxy-Cache

- Api will cache for faster response
- if one client changes data in server at the same time another cliet gets it, he wont get updated data
- it works on the based  on content type has to be cached in configuration
- you can test by looking at logs of your server, client will get reponse but server wont log it bcz Api itself responded request dint reached server
- change in config/kong.yml
```
- name: proxy-cache
  config: 
    response_code:
    - 200
    request_method:
    - GET
    - HEAD
    content_type:
    - text/plain
    - application/json
    - application/json; charset=utf-8
    cache_ttl: 300
    strategy: memory
  enabled: true
```	
Reference: https://docs.konghq.com/hub/kong-inc/proxy-cache/

########################################################
### Bsic-authentication,ACL
- we need consumers to activate it
- to access api using curl or  postman  we need to pass authorization header(we will get it from webpage,inspect,headers)ex: ization: Basic QWxhZGRpbjpPcGVhJKSGCSKO
```
 $curl http://100.64.188.202:8000/params
{
  "message":"Unauthorized"
}
```
correct command is 
```
curl-H 'authorization: Basic QWxhZGRpbjpPcGVuUXJHGDFYUG' http://100.64.188.202:8000/params
```
similarly you can pass in postman
- change config/kong.yml for authentication with username and passwd
```
consumers:
- username: user

basicauth_credentials:
- consumer: user
  username: Naveen
  password: Devops
  
plugins:  
- name: basic-auth
  config: 
    hide_credentials: true
  enabled: true
```
- we can pass a header with value to access api(ex.,apikey: abc)-->(better than previous) 
below command is used when both user and key is enabled in authorization
```
 curl -H 'authorization: Basic QWxhZGRpbjpPcGVuU2VHJJl' http://100.64.188.202:8000/params?apikey=abc
```
- changes in config/kong.yml
```
keyauth_credentials:
- consumer: user
  key: abc
plugins:  
- name: key-auth
  config: 
    key_names:
    - apikey
    key_in_body: false
    key_in_header: true
    key_in_query: true
    hide_credentials: false
    run_on_preflight: true
  enabled: true  
```  
Reference: https://docs.konghq.com/hub/kong-inc/basic-auth/

#####################################################################
### ACL plugin
- to configure acl we need authentication plugin as prerequisites
- authorization for a group .here we tested gfor group in which consumer named user is there
- you can allow /deny particular group for access
- changes in config/kong.yml
```
acls:
- consumer: user
  group: group1
  
plugins:  
- name: acl
  config: 
    allow:
    - group1
    hide_groups_header: true
  enabled: false  
```  

access_url: http://100.64.188.202:8000/params?apikey=abc 
Reference: https://docs.konghq.com/hub/kong-inc/acl/

