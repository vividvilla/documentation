---
title: Authorization Logs
description: Learn how to read Pomerium authorization logs.
pagination_prev: null
lang: en-US
keywords: [pomerium, troubleshooting, auth, authorization, logs]
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Pomerium provides verbose logging to help users manage and troubleshoot their configurations. The amount of logs output can be overwhelming, so on this page we'll cover how to sort for and understand logs from the authorization service. These logs often provide the most helpful information when first configuring Pomerium.

## Find Logs

The command to display Pomerium's logs will depend on your installation method:

<Tabs groupId="stacks">
<TabItem value="daemon" label="System Daemon">

```bash
sudo journalctl -u pomerium.service
```

The `-f` flag can also be provided to show new logs as they are generated.

</TabItem>
<TabItem value="docker" label="Docker">

```bash
docker logs pomerium
```

The container name will depend on your configuration. The `-f` flag can also be provided to show new logs as they are generated.

</TabItem>
<TabItem value="k8s" label="Kubernetes">

```bash
kubectl logs -f deploy/pomerium-authorize
```

The deployment name will depend on your configuration. The `-f` flag can also be provided to show new logs as they are generated.

</TabItem>
</Tabs>

### Filter Logs

To filter log output to just the Authorize service logs, we can pipe (`|`) the output through `grep`:

<Tabs groupId="stacks">
<TabItem value="daemon" label="System Daemon">

```bash
sudo journalctl -u pomerium.service | grep '"service":"authorize"'
```

</TabItem>
<TabItem value="docker" label="Docker">

```bash
docker logs pomerium | grep '"service":"authorize"'
```

</TabItem>
<TabItem value="k8s" label="Kubernetes">

```bash
kubectl logs -f deploy/pomerium-authorize | grep '"service":"authorize"'
```

</TabItem>
</Tabs>

### Formatted Logs

To better parse the logs, you can install [jq](https://stedolan.github.io/jq/) and use another pipe to pass the content through it:

```bash
kubectl logs -f deploy/pomerium-authorize | grep '"service":"authorize"' | jq
{
  "level": "info",
  "service": "authorize",
  "request-id": "46747f58-...",
  "check-request-id": "7700fe70-a166-406c-a326-48f433029f98",
  "method": "GET",
  "path": "/",
  "host": "grafana.example.com",
  "query": "",
  "session-id": "46b36e11...",
  "allow": false,
  "allow-why-false": ["claim-unauthorized", "non-pomerium-route"],
  "deny": false,
  "deny-why-false": ["valid-client-certificate-or-none-required"],
  "user": "941b0719-...",
  "email": "alex@example.com",
  "databroker_server_version": ...,
  "databroker_record_version": ...,
  "time": "2022-05-16T18:18:55Z",
  "message": "authorize check"
}
```

## Reading Authorization Logs (list)

The keys described below usually contain the relevant information when debugging an authorization issue:

- `request-id`: An id generated for every request that can be used to correlate logs.

- `allow`: If true, at least one allow rule passed. As long as `deny` is false, the request will be allowed.

- `allow-why-false` | `allow-why-true`: The short reason strings why access was allowed (or not allowed).
  
  In The example output above, `claim-unauthorized` means that there was a policy rule on a claim that didn't pass.
  
  `non-pomerium-route` means that it was not a request to `/.pomerium/`, which is  allowed for any authenticated user.

- `deny`: If true it means that at least one deny rule passed, and the request will be denied.

- `deny-why-false` | `deny-why-true`: The short reason strings why access was denied (or not denied).

  In the example above, `valid-client-certificate-or-none-required` means that either a valid client certificate was provided, or the policy didn't require one. By default pomerium policies have this PPL rule added to them. It's how we implement client certificates.

- `databroker_server_version` and `databroker_record_version`: These values are used for auditing. With these version numbers and a complete history of all changes in the databroker, you can determine what data was used for policy evaluation.

-----

## Reading Authorization Logs (Headers)

The keys described below usually contain the relevant information when debugging an authorization issue:

### `request-id`

An id generated for every request that can be used to correlate logs.

### `allow`

If true, at least one allow rule passed. As long as `deny` is false, the request will be allowed.

### `allow-why-false` | `allow-why-true`

The short reason strings why access was allowed (or not allowed).
  
In The example output above, `claim-unauthorized` means that there was a policy rule on a claim that didn't pass.
  
`non-pomerium-route` means that it was not a request to `/.pomerium/`, which is  allowed for any authenticated user.

### `deny`

If true it means that at least one deny rule passed, and the request will be denied.

### `deny-why-false` | `deny-why-true`

The short reason strings why access was denied (or not denied).

  In the example above, `valid-client-certificate-or-none-required` means that either a valid client certificate was provided, or the policy didn't require one. By default pomerium policies have this PPL rule added to them. It's how we implement client certificates.

### `databroker_server_version` | `databroker_record_version`

These values are used for auditing. With these version numbers and a complete history of all changes in the databroker, you can determine what data was used for policy evaluation.

-----

## Reading Authorization Logs (Table)

The keys described below usually contain the relevant information when debugging an authorization issue:

| Key                                                       | Description                                                                                                                                                                                                                                                                                                                                        |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `allow`                                                   | If true, at least one allow rule passed. As long as `deny` is false, the request will be allowed.                                                                                                                                                                                                                                                  |
| `allow-why-false` & `allow-why-true`                      | The short reason strings why access was allowed (or not allowed). <br/> <br/> In The example output above, `claim-unauthorized` means that there was a policy rule on a claim that didn't pass.<br/><br/>`non-pomerium-route` means that it was not a request to `/.pomerium/`, which is  allowed for any authenticated user.                      |
| `deny`                                                    | If true it means that at least one deny rule passed, and the request will be denied.                                                                                                                                                                                                                                                               |
| `deny-why-false` & `deny-why-true`                        | The short reason strings why access was denied (or not denied). <br/><br/>In the example above, `valid-client-certificate-or-none-required` means that either a valid client certificate was provided, or the policy didn't require one. By default pomerium policies have this PPL rule added to them. It's how we implement client certificates. |
| `databroker_server_version` & `databroker_record_version` | These values are used for auditing. With these version numbers and a complete history of all changes in the databroker, you can determine what data was used for policy evaluation.                                                                                                                                                                |
|                                                           |                                                                                                                                                                                                                                                                                                                                                    |