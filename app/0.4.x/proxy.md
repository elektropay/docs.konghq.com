---
title: Proxy Reference
---

# Proxy Reference

As you might already know, Kong uses two ports to communicate. By default they are:

`:8001` - The one on which the [Admin API][API] listens.

`:8000` - Where Kong listens for incoming requests to proxy to your upstream services. This is the port that interests us; here is a typical request workflow on this port:

<br />

![](/assets/images/docs/kong-simple.png)

<br />

This guide will cover all proxying capabilities of Kong by explaining in detail how the proxying (`8000`) port works under the hood.

---

## 1. How does Kong route a request to an API

When receiving a request, Kong will inspect it and try to route it to the correct API. In order to do so, it supports different routing mechanisms depending on your needs. A request can be routed by:

- A **DNS** value contained in the **Host** header of the request.
- The path (**URI**) of the request.

<div class="alert alert-warning">
  <strong>Note:</strong> Bear in mind that those two methods do not combine themselves together. It is currently <b>not</b> possible to define rules such as "with this hostname" <i>and</i> "with this URI prefix". Kong will first try to route a request by DNS and if unsuccessful, will try by URI.
</div>

<div class="alert alert-warning">
  <strong>Note:</strong> For performance reasons, Kong keeps a cache of the APIs from your Cassandra cluster in memory for up to 60 seconds. As cache invalidation has not been implemented yet, Kong might take up to <strong>60 seconds</strong> to notice a new API and proxy incoming requests to it.
</div>

---

## 2. Reminder: how to add an API to Kong

Before going any further, let's take a few moments to make sure you know how to add an API to Kong. This will also help clarify the difference between the two ports.

As explained in the [Adding your API][adding-your-api] quickstart guide, Kong is configured via its internal [Admin API][API] running by default on port `8001`. Adding an API to Kong is as easy as an HTTP request:

```bash
$ curl -i -X POST \
  --url http://localhost:8001/apis/ \
  -d 'name=mockbin' \
  -d 'target_url=http://mockbin.com/' \
  -d 'public_dns=mockbin.com' \
  -d 'path=/status'
```

This request tells Kong to add an API named "**mockbin**", with its upstream resource being located at "**http://mockbin.com**". The `public_dns` and `path` properties are the ones used by Kong to route a request to that API. Both properties are not required but at least one must be specified.

Once this request is processed by Kong, the API is stored in your Cassandra cluster and a request to the **Proxy port** will trigger a query to Cassandra and put your API in Kong's proxying cache.

---

## 3. Proxy an API by its request_host value

### Using the "**Host**" header

Now that we added an API to Kong (via the Admin API), Kong can proxy it via the `8000` port. One way to do so is to specify the API's `public_dns` value in the `Host` header of your request:

```bash
$ curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: mockbin.com'
```

By doing so, Kong recognizes the `Host` value as being the `public_dns` of the "mockbin" API. The request will be routed to the upstream API and Kong will execute any configured [plugin][plugins] for that API.

<div class="alert alert-warning">
  <strong>Going to production:</strong> If you're planning to go into production with your setup, you'll most likely not want your consumers to manually set the "<strong>Host</strong>" header on each request. You can let Kong and DNS take care of it by simply setting an A or CNAME record on your domain pointing to your Kong installation. Hence, any request made to `example.org` will already contain a `Host: example.org` header.
</div>

### Using the "**X-Host-Override**" header

When performing a request from a browser, you might not be able to set the `Host` header. Thus, Kong also checks a request for a header named `X-Host-Override` and treats it exactly like the `Host` header:

```bash
$ curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'X-Host-Override: mockbin.com'
```

This request will be proxied just as well by Kong.

### Using a wildcard DNS

Sometimes you might want to route all requests matching a wildcard DNS to your upstream services. A "**public_dns**" wildcard name may contain an asterisk only on the name’s start or end, and only on a dot border.

A "**public_dns**" of form `*.example.org` will route requests with "**Host**" values such as `a.example.org` or `x.y.example.org`.

A "**public_dns**" of form `example.*` will route requests with "**Host**" values such as `example.com` or `example.org`.

---

## 4. Proxy an API by its path value

If you'd rather configure your APIs so that Kong routes incoming requests according to the request's URI, Kong can also perform this function. This allows your consumers to seamlessly consume APIs sparing the headache of setting DNS records for your domains.

Because the API we previously configured has a `path` property, the following request will **also** be proxied to the upstream "mockbin" API:

```bash
$ curl -i -X GET \
  --url http://localhost:8000/status/200
```

You will notice this command makes a request to `KONG_URL:PROXY_PORT/status/200`. Since the configured `target_url` is `http://mockbin.com/`, the request will hit the upstream service at `http://mockbin.com/status/200`.

### Using the "**strip_path**" property

By enabling the `strip_path` property on an API, the requests will be proxied without the `path` property being included in the upstream request. Let's enable this option by making a request to the Admin API:

```bash
$ curl -i -X PATCH \
  --url http://localhost:8001/apis/mockbin \
  -d 'strip_path=true' \
  -d 'path=/mockbin'
```

Now that we slightly updated our API (you might have to wait a few seconds for Kong's proxying cache to be updated), Kong will proxy requests made to `KONG_URL:PROXY_PORT/mockbin` but will not include the `/mockbin` part when performing the upstream request.

Here is a table documenting the behaviour of the path routing depending on your API's configuration:

`path`      | `strip_path`   | incoming request       | upstream request
---         | ---            | ---                    | ---
`/mockbin`  | **false**      | `/some_path`           | **not proxied**
`/mockbin`  | **false**      | `/mockbin`             | `/mockbin`
`/mockbin`  | **false**      | `/mockbin/some_path`   | `/mockbin/some_path`
`/mockbin`  | **true**       | `/some_path`           | **not proxied**
`/mockbin`  | **true**       | `/mockbin`             | `/`
`/mockbin`  | **true**       | `/mockbin/some_path`   | `/some_path`

<div class="alert alert-warning">
  <strong>Note:</strong> The use of <code>strip_request_path</code> is irrelevant if the request is being routed using the request_host method (which has priority over the request_path routing). The request's URI will be left untouched.
</div>

---

## 5. Plugins execution

Once Kong has recognized which API an incoming request should be proxied to, it will look into your Cassandra cluster for any record of a [Plugin Configuration][plugin-configuration-object] for that particular API. This is done according to the following steps:

- 1. Kong recognized the API (according to one of the previously explained methods)
- 2. It looks into the datastore for Plugin Configurations for that API
- 3. Some Plugin Configurations were found, for example:
  - a. A key authentication Plugin Configuration
  - b. A ratelimiting Plugin Configuration (that also has a `consumer_id` property)
- 4. Kong executes the highest priority plugin (key authentication in this case)
  - a. User is now authenticated
- 5. Kong tries to execute the ratelimiting plugin
  - a. If the user is the one in the `consumer_id`, ratelimiting is applied
  - b. If the user is not the one configured, ratelimiting is not applied
- 6. Request is proxied

**Note**: The proxying of a request might happen before or after plugins execution, since each plugin can hook itself anywhere in the lifecycle of a request. In this case (authentication + ratelimiting) it is of course mandatory those plugins be executed **before** proxying happens.

[adding-your-api]: /{{page.kong_version}}/getting-started/adding-your-api
[API]: /{{page.kong_version}}/admin-api
[plugin-configuration-object]: /{{page.kong_version}}/admin-api#plugin-configuration-object
[plugins]: /plugins/
