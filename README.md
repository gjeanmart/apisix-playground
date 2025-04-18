# APISIX Exploration

## Requirements

- Docker
- Docker Compose
- ACD
  ```shell
  $ curl -sL "https://run.api7.ai/adc/install" | sh
  ```

## Getting started

1. **Start the Services**

Run the project:

```shell
docker compose up
```

This will start:

- `etcd` Key-Value store used by APISX
- `apisix` The API Gateway
- `apisix-dashboard:` Web UI to manage APISIX
- `httpbin` Sample HTTP service
- `redis` Used for rate limiting with persistence

=> Access the dashboard at: http://localhost:9000 (admin/admin)

2. **Export your API Admin key**

Find the Admin API key in `api.config.yaml` under `deployment.admin.admin_key`, and export it:

```shell
export ADMIN_KEY=adminkey
```

## Configure Upstream and route

We’ll start by routing requests to the `httpbin` upstream.

**Via UI**

![](./docs//dashboard_upstream_create.png)

![](./docs/dashboard_route_create_1.png)

![](./docs/dashboard_route_create_2.png)

![](./docs/dashboard_route_create_3.png)

**Via Admin API**

```shell
$ curl http://127.0.0.1:9180/apisix/admin/routes/1 -H "X-API-KEY: $ADMIN_KEY" -X PUT -i -d '
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

TBC

**Testing**

Test the Route

```shell
$ curl http://localhost:9080/httpbin/ip  -i
HTTP/1.1 200 OK
{ "origin": "192.168.65.1" }
```

## API Key Auth + Rate Limiting

We’ll now enable API key-based authentication and configure rate limits based on consumer groups: `basic` and `premium`.

We will be using the following plugins

- `key-auth`: https://apisix.apache.org/docs/apisix/plugins/key-auth/
- `limit-req`: https://apisix.apache.org/docs/apisix/plugins/limit-req/

### Create Consumer Groups

First we create Consumer Groups for each plan (basic and premium) using the plugin `limit-req`

**Via UI**

Not possible

**Via Admin API**

We'll define two plans with different rate limits:

Basic Plan (1 request/second)

```shell
$ curl http://127.0.0.1:9180/apisix/admin/consumer_groups/basic_plan -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
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

Premium Plan (10 requests/second)

```shell
$ curl http://127.0.0.1:9180/apisix/admin/consumer_groups/premium_plan -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
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

### Create Consumers

Let's create two consumers, one for each consumer group/plan leveraging `key-auth` plugin

**Via UI**

Not possible with plugins/group_id

**Via Admin API**

```shell
$ curl http://127.0.0.1:9180/apisix/admin/consumers -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
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
$ curl http://127.0.0.1:9180/apisix/admin/consumers -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
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

TBC

### Update the Route

Finally, let's update Route to enable Auth

```shell
$ curl http://127.0.0.1:9180/apisix/admin/routes/1 -H "X-API-KEY: $ADMIN_KEY" -X PUT -i -d '
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

Try reaching out the service through the API Gateway

```shell
$ curl http://localhost:9080/httpbin/ip  -i
HTTP/1.1 401 Unauthorized
{"message":"Missing API key in request"}
```

```shell
$ curl http://localhost:9080/httpbin/ip  -i -H 'apikey: apikey1'
HTTP/1.1 200 OK
{"origin": "192.168.65.1"}
```

```shell
$ curl http://localhost:9080/httpbin/ip  -i -H 'apikey: apikey1'
HTTP/1.1 429 Too Many Requests
<html>
<head><title>429 Too Many Requests</title></head>
<body>
<center><h1>429 Too Many Requests</h1></center>
<hr><center>openresty</center>
<p><em>Powered by <a href="https://apisix.apache.org/">APISIX</a>.</em></p></body>
</html>
```

## Configure API Authentication with Long-Lived Rate Limits

We'll now enforce monthly quotas using `limit-count` (https://apisix.apache.org/docs/apisix/plugins/limit-count/)

**Via UI**

Not possible

**Via Admin API**

Update the **Basic Plan** to 10 requests per month:

```shell
$ curl http://127.0.0.1:9180/apisix/admin/consumer_groups/basic_plan -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
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

Update the **Premium Plan** limited to 10000 request per month

```shell
$ curl http://127.0.0.1:9180/apisix/admin/consumer_groups/premium_plan -H "X-API-KEY: $ADMIN_KEY" -X PUT -d '
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

TBC

**Testing**
Try out to reach the service through the API Gateway

```shell
$ curl http://localhost:9080/httpbin/ip  -i -H 'apikey: apikey1'
HTTP/1.1 200 OK
{"origin": "192.168.65.1"}
```

After exceeding monthly quota (x10):

```shell
$ curl http://localhost:9080/httpbin/ip  -i -H 'apikey: apikey1'
HTTP/1.1 429 Too Many Requests
<html>
<head><title>429 Too Many Requests</title></head>
<body>
<center><h1>429 Too Many Requests</h1></center>
<hr><center>openresty</center>
<p><em>Powered by <a href="https://apisix.apache.org/">APISIX</a>.</em></p></body>
</html>
```

## Metrics

TBC
