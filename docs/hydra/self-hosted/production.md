---
id: production
title: Prepare for production
---

Read this document to prepare for production when self-hosting Ory Hydra.  
Feel free to [open an issue or pull request](https://github.com/ory/docs/) when you have an idea how to improve this
documentation.

Read more about [deployment fundamentals and requirements for Ory](../../self-hosted/deployment).

## Ory Hydra behind an API gateway

Although Ory Hydra implements all Go best practices around running public-facing production HTTP servers, we discourage running
Ory Hydra facing the public net directly. We strongly recommend running Ory Hydra behind an API gateway or a load balancer. It's
common to terminate TLS on the edge (gateway / load balancer) and use certificates provided by your infrastructure provider such
as AWS CA for last mile security.

### HTTP clients

In some scenarios you might want to disallow HTTP calls to private IP ranges. To configure this feature, set the following
configuration:

```yaml
clients:
  http:
    disallow_private_ip_ranges: true
```

If enabled, all outgoing HTTP calls done by Ory Hydra will be checked whether they're against a private IP range. If that's the
case, the request will fail with an error.

### Routing

It's common to use a router, or API gateway, to route subdomains or paths to a specific service. For example,
`https://myservice.com/hydra/` is routed to `http://10.0.1.213:3912/` where `10.0.1.213` is the host running Ory Hydra. To compute
the values for the consent challenge, Ory Hydra uses the host and path headers from the HTTP request. Therefore, it's important to
set up your API Gateway in such a way, that it passes the public host (in this case `myservice.com`) and the path without any
prefix (in this case `hydra/`). If you use the Mashape Kong API gateway, you can achieve this by setting `strip_request_path=true`
and `preserve_host=true.`

## Exposing administrative and public API endpoints

Ory Hydra serves APIs via two ports:

- Public port (default 4444)
- Administrative port (default 4445)

The public port can and should be exposed to public internet traffic. That port handles requests to:

- `/.well-known/jwks.json`
- `/.well-known/openid-configuration`
- `/oauth2/auth`
- `/oauth2/token`
- `/oauth2/revoke`
- `/oauth2/fallbacks/consent`
- `/oauth2/fallbacks/error`
- `/oauth2/sessions/logout`
- `/userinfo`

The administrative port shouldn't be exposed to public internet traffic. If you want to expose certain endpoints, such as the
`/clients` endpoint for OpenID Connect Dynamic Client Registry, you can do so but you need to properly secure these endpoints with
an API Gateway or Authorization Proxy. Administrative endpoints include:

- All `/clients` endpoints.
- All `/keys` endpoints.
- All `/health`, `/admin/metrics/prometheus`, `/admin/version` endpoints.
- All `/oauth2/auth/requests` endpoints.
- Endpoint `/oauth2/introspect`.
- Endpoint `/oauth2/flush`.

None of the administrative endpoints have any built-in access control. You can do simple `curl` or Postman requests to talk to
them.

The Token Introspection endpoint requires authentication. But since there is no access control, any valid authentication enables
the endpoint to be used. If you need to access this endpoint in production, you should configure your API Gateway or Application
Proxy to restrict which clients have access to the endpoint.

We generally advise to run Ory Hydra with `hydra serve all` which listens on both ports in one process.

### Binding to different interfaces or UNIX sockets

Ory Hydra will bind public and administrative APIs ports to all interfaces.

The interfaces or UNIX sockets used may be specified via environment variables `PUBLIC_HOST` and `ADMIN_HOST`. Interfaces may be
specified as TCP address or as UNIX socket (giving the absolute path to the socket file prefixed by `unix:`) like:

- `PUBLIC_HOST=127.0.0.1`
- `ADMIN_HOST="unix:/var/run/hydra/admin_socket"`

Ory Hydra will try to create the socket file during startup and the socket will be writeable by the user running Ory Hydra. The
owner, group and mode of the socket can be modified:

```yaml
serve:
  admin:
    host: unix:/var/run/hydra/admin_socket
    socket:
      owner: hydra
      group: hydra-admin-api
      mode: 770
```

### Key generation and high availability environments

Be aware that on the very first launch of the Hydra container(s), a worker process will perform certain first-time installation
tasks, such as generating [JSON web keys](../jwks) if they don't already exist.

If you intend on running your production Hydra environment in a highly-available setup (for example, multiple concurrent
containers behind a load-balancer), it's possible that both containers will generate JWKs at the same time.

Although this isn't a problem, we recommend that you launch your production environment with just one container to begin with, to
complete the initial seeding of the database.

Once done, you can raise your number of containers to achieve high availability.

## Next steps

For a deployment guide using Nginx, visit the [Deploy to production](./deploy-hydra-example) documentation.
