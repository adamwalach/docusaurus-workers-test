---
id: authz
title: Authorizers
---

An "authorizer" is responsible for properly permissioning a subject. Ory Oathkeeper supports different kinds of authorizers. The
list of authorizers increases over time due to new features and requirements.

Authorizers assure that a subject, for instance a "user", has the permissions necessary to access or perform a particular service.
For example, an authorizer can permit access to an endpoint or URL for specific subjects or "users" from a specific group "admin".
The authorizer permits the subjects the desired access to the endpoint.

Each authorizer has two keys:

- `handler` (string, required): Defines the handler, for example `noop`, to be used.
- `config` (object, optional): Configures the handler. Configuration keys can vary for each handler.s

```json
{
  "authorizer": {
    "handler": "noop",
    "config": {}
  }
}
```

There is a 1:1 mandatory relationship between an authoriser and an access rule. It isn't possible to configure more than one
authorizer per Access Rule.

## `allow`

This authorizer permits every action allowed.

### `allow` configuration

This handler isn't configurable.

To enable this handler, set as follows:

```yaml
# Global configuration file oathkeeper.yml
authorizers:
  allow:
    # Set enabled to "true" to enable the authenticator, and "false" to disable the authenticator. Defaults to "false".
    enabled: true
```

### `allow` access rule example

```sh
$ cat ./rules.json

[{
  "id": "some-id",
  "upstream": {
    "url": "http://my-backend-service"
  },
  "match": {
    "url": "http://my-app/some-route",
    "methods": [
      "GET"
    ]
  },
  "authenticators": [{ "handler": "anonymous" }],
  "authorizer": { "handler": "allow" },
  "mutators": [{ "handler": "noop" }]
}]

$ curl -X GET http://my-app/some-route

HTTP/1.0 200 Status OK
The request has been allowed!
```

## `deny`

This authorizer considers every action unauthorized therefore "forbidden" or "disallowed".

### `deny` configuration

This handler isn't configurable.

To enable this handler, set:

```yaml
# Global configuration file oathkeeper.yml
authorizers:
  deny:
    # Set enabled to "true" to enable the authenticator, and "false" to disable the authenticator. Defaults to "false".
    enabled: true
```

### `deny` access rule example

```sh
$ cat ./rules.json

[{
  "id": "some-id",
  "upstream": {
    "url": "http://my-backend-service"
  },
  "match": {
    "url": "http://my-app/some-route",
    "methods": [
      "GET"
    ]
  },
  "authenticators": [{ "handler": "anonymous" }],
  "authorizer": { "handler": "deny" },
  "mutators": [{ "handler": "noop" }]
}]

$ curl -X GET http://my-app/some-route

HTTP/1.0 403 Forbidden
The request is forbidden!
```

## `keto_engine_acp_ory`

This authorizer uses the Ory Keto API to carry out access control using "Ory-flavored" Access Control Policies. The conventions
used in the Ory Keto project are located on [GitHub Ory Keto](https://github.com/ory/keto) for consultation prior to using this
authorizer.

### `keto_engine_acp_ory` configuration

- `base_url` (string, required) - The base URL of Ory Keto, typically something like `https://hostname:port/`
- `required_action` (string, required) - See section below.
- `required_resource` (string, required) - See section below.
- `subject` (string, optional) - See section below.
- `flavor` (string, optional) - See section below.

#### Resource, relation, subject

Actions were renamed to relations. Read the
[Ory Keto policy migration guide](../../keto/guides/migrating-legacy-policies#rewriting-it-to-relationships) for more details.

This authorizer has four configuration options, `required_action`, `required_resource`, `subject`, and `flavor`:

```json
{
  "handler": "keto_engine_acp_ory",
  "config": {
    "required_action": "...",
    "required_resource": "...",
    "subject": "...",
    "flavor": "..."
  }
}
```

All configuration options except `flavor` support Go [`text/template`](https://golang.org/pkg/text/template/). For example in the
following match configuration:

```json
{
  "match": {
    "url": "http://my-app/api/users/<[0-9]+>/<[a-zA-Z]+>",
    "methods": ["GET"]
  }
}
```

The following example shows how to reference the values matched by or resulting from the two regular expressions, `<[0-9]+>` and
`<[a-zA-Z]+>`. using the `AuthenticationSession` struct:

```json
{
  "handler": "keto_engine_acp_ory",
  "config": {
    "required_action": "my:action:{{ printIndex .MatchContext.RegexpCaptureGroups 0 }}",
    "required_resource": "my:resource:{{ printIndex .MatchContext.RegexpCaptureGroups 1 }}:foo:{{ printIndex .MatchContext.RegexpCaptureGroups 0 }}"
  }
}
```

Assuming a request to `http://my-api/api/users/1234/foobar` was made, the config from above would expand to:

```json
{
  "handler": "keto_engine_acp_ory",
  "config": {
    "required_action": "my:action:1234",
    "required_resource": "my:resource:foobar:foo:1234"
  }
}
```

The `subject` field configures the subject that passes to the Ory Keto endpoint. If `subject` isn't specified it will default to
`AuthenticationSession.Subject`.

For more details about supported Go template substitution, see. [How to use session variables](../pipeline.md#session)

#### `keto_engine_acp_ory` example

```yaml
# Global configuration file oathkeeper.yml
authorizers:
  keto_engine_acp_ory:
    # Set enabled to "true" to enable the authenticator, and "false" to disable the authenticator. Defaults to "false".
    enabled: true

    config:
      base_url: http://my-keto/
      required_action: ...
      required_resource: ...
      subject: ...
      flavor: ...
```

```yaml
# Some Access Rule: access-rule-1.yaml
id: access-rule-1
# match: ...
# upstream: ...
authorizers:
  - handler: keto_engine_acp_ory
    config:
      base_url: http://my-keto/
      required_action: ...
      required_resource: ...
      subject: ...
      flavor: ...
```

### `keto_engine_acp_ory` access rule example

```shell
$ cat ./rules.json

[{
  "id": "some-id",
  "upstream": {
    "url": "http://my-backend-service"
  },
  "match": {
    "url": "http://my-app/api/users/<[0-9]+>/<[a-zA-Z]+>",
    "methods": [
      "GET"
    ]
  },
  "authenticators": [
    {
      "handler": "anonymous"
    }
  ],
  "authorizer": {
    "handler": "keto_engine_acp_ory",
    "config": {
      "required_action": "my:action:$1",
      "required_resource": "my:resource:$2:foo:$1"
      "subject": "{{ .Extra.email }}",
      "flavor": "exact"
    }
  }
  "mutators": [
    {
      "handler": "noop"
    }
  ]
}]
```

## `remote`

This authorizer performs authorization using a remote authorizer. The authorizer makes a HTTP POST request to a remote endpoint
with the original body request as body. If the endpoint returns a "200 OK" response code, the access is allowed, if it returns a
"403 Forbidden" response code, the access is denied.

### `remote` configuration

- `remote` (string, required) - The remote authorizer's URL. The remote authorizer is expected to return either "200 OK" or "403
  Forbidden" to allow/deny access.
- `headers` (map of strings, optional) - The HTTP headers sent to the remote authorizer. The values will be parsed by the Go
  [`text/template`](https://golang.org/pkg/text/template/) package and applied to an
  [`AuthenticationSession`](https://github.com/ory/oathkeeper/blob/master/pipeline/authn/authenticator.go#L40) object. See
  [Session](../pipeline.md#session) for more details.
- `forward_response_headers_to_upstream` (slice of strings, optional) - The HTTP headers that will be allowed from remote
  authorizer responses. If returned, headers on this list will be forward to upstream services.
- `retry` (object, optional) - Configures timeout and delay settings for the request against the token endpoint
  - `give_up_after` (string) max delay duration of retry. The value will be parsed by the Go
    [duration parser](https://pkg.go.dev/time#ParseDuration).
  - `max_delay` (string) time to wait between retries and max service response time. The value will be parsed by the Go
    [duration parser](https://pkg.go.dev/time#ParseDuration).

#### `remote` example

```yaml
# Global configuration file oathkeeper.yml
authorizers:
  remote:
    # Set enabled to "true" to enable the authenticator, and "false" to disable the authenticator. Defaults to "false".
    enabled: true

    config:
      remote: http://my-remote-authorizer/authorize
      headers:
        X-Subject: "{{ print .Subject }}"
```

```yaml
# Some Access Rule: access-rule-1.yaml
id: access-rule-1
# match: ...
# upstream: ...
authorizers:
  - handler: remote
    config:
      remote: http://my-remote-authorizer/authorize
      headers:
        X-Subject: "{{ print .Subject }}"
```

### `remote` access rule example

```shell
{
  "id": "some-id",
  "upstream": {
    "url": "http://my-backend-service"
  },
  "match": {
    "url": "http://my-app/api/<.*>",
    "methods": ["GET"]
  },
  "authenticators": [
    {
      "handler": "anonymous"
    }
  ],
  "authorizer": {
    "handler": "remote",
    "config": {
      "remote": "http://my-remote-authorizer/authorize",
      "headers": {
        "X-Subject": "{{ print .Subject }}"
      },
      "forward_response_headers_to_upstream": [
        "X-Foo"
      ]
    }
  }
  "mutators": [
    {
      "handler": "noop"
    }
  ]
}
```

## `remote_json`

This authorizer performs authorization using a remote authorizer. The authorizer makes a HTTP POST request to a remote endpoint
with a JSON body. If the endpoint returns a "200 OK" response code, the access is allowed, if it returns a "403 Forbidden"
response code, the access is denied.

### `remote_json` configuration

- `remote` (string, required) - The remote authorizer's URL. The remote authorizer is expected to return either "200 OK" or "403
  Forbidden" to allow/deny access.
- `payload` (string, required) - The request's JSON payload sent to the remote authorizer. The string will be parsed by the Go
  [`text/template`](https://golang.org/pkg/text/template/) package and applied to an
  [`AuthenticationSession`](https://github.com/ory/oathkeeper/blob/master/pipeline/authn/authenticator.go#L40) object. See
  [Session](../pipeline.md#session) for more details.
- `headers` (map of strings, optional) - The HTTP headers sent to the remote authorizer. The values will be parsed by the Go
  [`text/template`](https://golang.org/pkg/text/template/) package and applied to an
  [`AuthenticationSession`](https://github.com/ory/oathkeeper/blob/master/pipeline/authn/authenticator.go#L40) object. See
  [Session](../pipeline.md#session) for more details.
- `forward_response_headers_to_upstream` (slice of strings, optional) - The HTTP headers that will be allowed from remote
  authorizer responses. If returned, headers on this list will be forward to upstream services.
- `retry` (object, optional) - Configures timeout and delay settings for the request against the token endpoint
  - `give_up_after` (string) max delay duration of retry. The value will be parsed by the Go
    [duration parser](https://pkg.go.dev/time#ParseDuration).
  - `max_delay` (string) time to wait between retries and max service response time. The value will be parsed by the Go
    [duration parser](https://pkg.go.dev/time#ParseDuration).

#### `remote_json` example

```yaml
# Global configuration file oathkeeper.yml
authorizers:
  remote_json:
    # Set enabled to "true" to enable the authenticator, and "false" to disable the authenticator. Defaults to "false".
    enabled: true

    config:
      remote: http://my-remote-authorizer/authorize
      headers:
        Y-Api-Key: '{{ .MatchContext.Header.Get "X-Api-Key" }}'
      payload: |
        {
          "subject": "{{ print .Subject }}",
          "resource": "{{ printIndex .MatchContext.RegexpCaptureGroups 0 }}"
        }
```

```yaml
# Some Access Rule: access-rule-1.yaml
id: access-rule-1
# match: ...
# upstream: ...
authorizers:
  - handler: remote_json
    config:
      remote: http://my-remote-authorizer/authorize
      headers:
        Y-Api-Key: '{{ .MatchContext.Header.Get "X-Api-Key" }}'
      payload: |
        {
          "subject": "{{ print .Subject }}",
          "resource": "{{ printIndex .MatchContext.RegexpCaptureGroups 0 }}"
        }
```

### `remote_json` access rule example

```shell
{
  "id": "some-id",
  "upstream": {
    "url": "http://my-backend-service"
  },
  "match": {
    "url": "http://my-app/api/<.*>",
    "methods": ["GET"]
  },
  "authenticators": [
    {
      "handler": "anonymous"
    }
  ],
  "authorizer": {
    "handler": "remote_json",
    "config": {
      "headers": {
         "Y-Api-Key": "{{ .MatchContext.Header.Get \"X-Api-Key\" }}"
      },
      "remote": "http://my-remote-authorizer/authorize",
      "payload": "{\"subject\": \"{{ print .Subject }}\", \"resource\": \"{{ printIndex .MatchContext.RegexpCaptureGroups 0 }}\"}"
    },
    "forward_response_headers_to_upstream": [
      "X-Foo"
    ]
  }
  "mutators": [
    {
      "handler": "noop"
    }
  ]
}
```
