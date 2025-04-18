## APISIX Exploration

### Requirements

- Docker
- Docker Compose
- ACD
  ```shell
  curl -sL "https://run.api7.ai/adc/install" | sh
  ```

### Getting started

```shell
docker compose up
```

Go to http://localhost:9000

Export your API Admin key (find it in `api.apisix.yaml` -> `deployment.admin.admin_key`)

```shell
export ADMIN_KEY=adminkey
```

#### Configure Upstream and route

**Via UI**

<TODO>

**Via Admin API**

```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 -H "X-API-KEY: $ADMIN_KEY" -X PUT -i -d '
{
    "name": "httbin-route",
    "uri": "/httpbin/*",
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "httpbin:80": 1
        }
    },
    "plugins": {
        "proxy-rewrite": {
            "regex_uri": ["/httpbin/(.*)", "/$1"]
        }
    }
}'
```

**Via ACD**

<TODO>

**Testing**
Try out to reach the service through the API Gateway

```shell
curl http://localhost:9080/httpbin/ip  -i
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 31
Connection: keep-alive
Date: Fri, 18 Apr 2025 10:07:45 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.12.0

{
  "origin": "192.168.65.1"
}
```

#### Configure API Authentication with per-second rate limits

- key-auth: https://apisix.apache.org/docs/apisix/plugins/key-auth/
- limit-req: https://apisix.apache.org/docs/apisix/plugins/limit-req/

##### Consumer groups

**Via UI**

No possible

**Via Admin API**

Create a **Basic Plan** limited to 1 request per secod

```shell
curl http://127.0.0.1:9180/apisix/admin/consumer_groups/basic_plan -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
{
    "plugins": {
        "limit-req": {
            "rate": 1,
            "burst": 0,
            "key": "consumer_name",
            "rejected_code": 429
        }
    }
}'
```

Create a **Premium Plan** limited to 200 requests per minute

```shell
curl http://127.0.0.1:9180/apisix/admin/consumer_groups/premium_plan -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
{
    "plugins": {
        "limit-req": {
            "rate": 10,
            "burst": 0,
            "key": "consumer_name",
            "rejected_code": 429
        }
    }
}'
```

**Via ACD**

<TODO>

##### Consumers

**Via UI**

<TODO>

**Via Admin API**

```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
{
    "username": "consumer1",
    "plugins": {
        "key-auth": {
            "key": "apikey1"
        }
    },
    "group_id": "basic_plan"
}'
```

```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
{
    "username": "consumer2",
    "plugins": {
        "key-auth": {
            "key": "apikey2"
        }
    },
    "group_id": "premium_plan"
}'
```

**Via ACD**

<TODO>

##### Update the route

```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 -H "X-API-KEY: $ADMIN_KEY" -X PUT -i -d '
{
    "name": "httbin-route",
    "uri": "/httpbin/*",
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "httpbin:80": 1
        }
    },
    "plugins": {
        "proxy-rewrite": {
            "regex_uri": ["/httpbin/(.*)", "/$1"]
        },
        "key-auth": {}
    }
}'
```

**Testing**
Try out to reach the service through the API Gateway

```shell
curl http://localhost:9080/httpbin/ip  -i
HTTP/1.1 401 Unauthorized
Date: Fri, 18 Apr 2025 12:35:08 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.12.0

{"message":"Missing API key in request"}
```

```shell
curl http://localhost:9080/httpbin/ip  -i -H 'apikey: apikey1'
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 31
Connection: keep-alive
Date: Fri, 18 Apr 2025 12:51:47 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.12.0

{
  "origin": "192.168.65.1"
}
```

```shell
sandbox curl http://localhost:9080/httpbin/ip  -i -H 'apikey: apikey1'
HTTP/1.1 429 Too Many Requests
Date: Fri, 18 Apr 2025 12:51:48 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 241
Connection: keep-alive
Server: APISIX/3.12.0

<html>
<head><title>429 Too Many Requests</title></head>
<body>
<center><h1>429 Too Many Requests</h1></center>
<hr><center>openresty</center>
<p><em>Powered by <a href="https://apisix.apache.org/">APISIX</a>.</em></p></body>
</html>
```

#### Configure API Authentication with long-live rate limits

- limit-count: https://apisix.apache.org/docs/apisix/plugins/limit-count/

**Via UI**

No possible

**Via Admin API**

Update the **Basic Plan** limited to 10 request per month

```shell
curl http://127.0.0.1:9180/apisix/admin/consumer_groups/basic_plan -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
{
    "plugins": {
        "limit-req": {
            "rate": 1,
            "burst": 0,
            "key": "consumer_name",
            "rejected_code": 429
        },
        "limit-count": {
            "count": 10,
            "time_window": 2592000,
            "key": "consumer_name",
            "policy": "redis",
            "redis_host": "redis",
            "redis_port": 6379,
            "rejected_code": 429
        }
    }
}'
```

Update the **Basic Plan** limited to 10000 request per month

```shell
curl http://127.0.0.1:9180/apisix/admin/consumer_groups/premium_plan -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
{
    "plugins": {
        "limit-req": {
            "rate": 10,
            "burst": 0,
            "key": "consumer_name",
            "rejected_code": 429
        },
        "limit-count": {
            "count": 10000,
            "time_window": 2592000,
            "key": "consumer_name",
            "policy": "redis",
            "redis_host": "redis",
            "redis_port": 6379,
            "rejected_code": 429
      }
    }
}'
```

**Via ACD**

<TODO>

**Testing**
Try out to reach the service through the API Gateway

```shell
curl http://localhost:9080/httpbin/ip  -i
HTTP/1.1 401 Unauthorized
Date: Fri, 18 Apr 2025 12:35:08 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.12.0

{"message":"Missing API key in request"}
```

```shell
curl http://localhost:9080/httpbin/ip  -i -H 'apikey: apikey1'
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 31
Connection: keep-alive
Date: Fri, 18 Apr 2025 12:51:47 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.12.0

{
  "origin": "192.168.65.1"
}
```

x10

```shell
sandbox curl http://localhost:9080/httpbin/ip  -i -H 'apikey: apikey1'
HTTP/1.1 429 Too Many Requests
Date: Fri, 18 Apr 2025 12:51:48 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 241
Connection: keep-alive
Server: APISIX/3.12.0

<html>
<head><title>429 Too Many Requests</title></head>
<body>
<center><h1>429 Too Many Requests</h1></center>
<hr><center>openresty</center>
<p><em>Powered by <a href="https://apisix.apache.org/">APISIX</a>.</em></p></body>
</html>
```
