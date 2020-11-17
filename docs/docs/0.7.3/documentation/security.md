# Security

This section addresses the certificate generation in Kuma across the following use-cases:

1. Security between our services, via the [mTLS policy](/docs/0.7.3/policies/mutual-tls/).
2. Security between the Kuma control plane and its data plane proxies, via the data plane proxy token.
3. Security when accessing the control plane.

:::tip
This section is not to be confused with the [mTLS policy](/docs/0.7.3/policies/mutual-tls/) that we can apply to a [Mesh](/docs/0.7.3/policies/mesh/) to secure service-to-service traffic.
:::

## Certificates

In Kuma, any TLS certificate that is being issued by the control plane must have a CA (Certificate Authority) that can be either auto-generated by Kuma via a `builtin` backend, or can be initialized with a customer certificate and key via a `provided` backend.

:::tip
Third-party extensions, cloud implementations or [commercial offerings](/enterprise) may be extending the CA backend support.
::: 

For both `builtin` and `provided` CA backends, on Kubernetes the root CA certificate is stored as a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/), while on Universal Kuma leverages the same underlying storage backend that is used for storing policies.

:::tip
When the [mTLS policy](/docs/0.7.3/policies/mutual-tls/) is enabled, data plane proxy certificates are ephemeral: thet are re-created on every data plane proxy restart and never persisted on disk.
:::

Data plane proxy certificates generated by Kuma are X.509 certificates that are [SPIFFE](https://github.com/spiffe/spiffe/blob/master/standards/X509-SVID.md) compliant. The SAN of the certificates is set to `spiffe://<mesh name>/<service name>`.

## Data plane proxy token

The data plane proxy token comes into play when it comes to security the communication between the data plane proxies and the control plane, that is between `kuma-dp` (including `envoy`) and `kuma-cp`.

In order to obtain an mTLS certificate from the server ([SDS](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret) built-in in the control plane), a data plane proxy must first prove its identity.

### Kubernetes

On Kubernetes, a data plane proxy proves its identity by leveraging the [ServiceAccountToken](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#service-account-automation) that is mounted in every pod.

### Universal

On Universal, a data plane proxy must be explicitly configured with a unique security token (data plane proxy token) that will be used to prove its identity. The data plane proxy token is a signed [JWT token](https://jwt.io) that carries the data plane proxy name and name of the mesh that it belongs to.

It is signed by an RSA key auto-generated by the control plane on first run. Tokens are never stored in the control plane,
the only thing that is stored is a signing key that is used to verify if a token is valid.

We can generate a token using the following methods:

:::: tabs :options="{ useUrlFragment: false }"
::: tab "kumactl"

We can generate a data plane proxy token using Kuma's CLI:

```bash
kumactl generate dataplane-token \
  --name dp-echo-1 \
  --mesh default \
  --type dataplane \
  --tag kuma.io/service=backend,backend-admin > /tmp/kuma-dp-echo1-token
``` 

:::
::: tab "HTTP API"

We can generate a token by making `POST` request to `:5679/tokens` on the control plane, like:

```bash
curl -XPOST \ 
  -H "Content-Type: application/json" \
  --data '{"name": "dp-echo-1", "mesh": "default", "type": "dataplane", tags": {"kuma.io/service": ["backend", "backend-admin"]}}' \
  http://control-plane:5679/tokens
```

:::
::::

The data plane proxy token should be stored in a file and then used when starting `kuma-dp`:

```bash
$ kuma-dp run \
  --name=dp-echo-1 \
  --mesh=default \
  --cp-address=http://127.0.0.1:5681 \
  --dataplane-token-file=/tmp/kuma-dp-echo-1-token
```

We can also pass the data plane proxy token as `KUMA_DATAPLANE_RUNTIME_TOKEN` environment variable. 

#### Data Plane Proxy Token boundary

As we can see in the example above, we can generate a token by passing a `name`, `mesh`, and a list of tags. The control plane will then verify the data plane proxy resources that are connecting to it against the token. This means we can generate a token by specifying:

* Only `mesh`. By doing so we can reuse the token for all dataplanes in a given mesh.
* `mesh` + `tag` (ex. `kuma.io/service`). This way we can use one token across all instances/replicas of the given service. Please keep in mind that we have to specify to include all the services that a data plane proxy is in charge of. For example, if we have a Dataplane with two inbounds, one valued with `kuma.io/service: backend` and one with `kuma.io/service: backend-admin`, we need to specify both values (`--tag kuma.io/service=backend,backend-admin`).
* `mesh` + `name` + `tag` (ex. `kuma.io/service`). This way we can use one token for one instance/replicate of the given service.
* `type`. The type can be either `dataplane` (default if not specified) or `ingress`. An [ingress data plane proxy](/docs/0.7.3/documentation/dps-and-data-model/#dataplane-specification) in a [multi-zone deployment](/docs/0.7.3/documentation/deployments/) requires an ingress token.

#### Disabling Data Plane Proxy Token

We can disable the security between the control plane and its data plane proxies by setting `KUMA_ADMIN_SERVER_APIS_DATAPLANE_TOKEN_ENABLED` to `false`.

::: warning
If we disable the security between the control plane and the data plane proxies, any data plane proxy will be able to impersonate any service, therefore this is not recommended in production.
:::

#### Accessing Admin Server from a different machine

By default, the Admin Server that is serving the data plane proxy tokens is exposed only on `localhost`. To generate tokens from a different machine than the one where the control plane is running (`kuma-cp`) we must first secure our connections:

1) Enable `kuma-cp` to accept requests from third party clients by setting `KUMA_ADMIN_SERVER_PUBLIC_ENABLED` to `true`. Please make sure to specify on what hostname we will make Kuma accessile to 3rd parties by setting the `KUMA_GENERAL_ADVERTISED_HOSTNAME` environment variable.
2) Generate a certificate for the HTTPS Admin Server and pass it to Kuma via the `KUMA_ADMIN_SERVER_PUBLIC_TLS_CERT_FILE` and `KUMA_ADMIN_SERVER_PUBLIC_TLS_KEY_FILE` config environment variables.

To generate a self-signed certificate we can use `kumactl`:

```sh
$ kumactl generate tls-certificate \
    --cert-file=/path/to/cert \
    --key-file=/path/to/key \
    --type=server \
    --cp-hostname=<value of KUMA_GENERAL_ADVERTISED_HOSTNAME>
```

3) Pick a public interface on which the HTTPS server will listen to, and set it via `KUMA_ADMIN_SERVER_PUBLIC_INTERFACE`. Optionally we can select the port via the `KUMA_ADMIN_SERVER_PUBLIC_PORT` environment variable. By default, it will be the same port as the one for the HTTP server exposed on `localhost`.
4) Generate one or more certificates for the clients that will access the control plane server. Then pass the path of the directory with the client certificates (without keys) via the `KUMA_ADMIN_SERVER_PUBLIC_CLIENT_CERTS_DIR` env variable.

For generating self signed client certificates we can use `kumactl`:

```sh
$ kumactl generate tls-certificate \
    --cert-file=/path/to/cert \
    --key-file=/path/to/key \
    --type=client
```

5) Configure `kumactl` with a valid client certificate:

```sh
$ kumactl config control-planes add \
  --name <NAME> --address http://<KUMA_CP_DNS_NAME>:5681 \
  --admin-client-cert <CERT.PEM> \
  --admin-client-key <KEY.PEM>
```

## mTLS

Once a data plane proxy has proved its identity, it will be allowed to fetch its own identity certificate and a root CA certificate of the mesh it belongs to. When establishing a connection between two dataplanes each side validates each other dataplane certificate confirming the identity using the root CA of the mesh.

mTLS is _not_ enabled by default. To enable it, apply proper settings in [Mesh](../../policies/mesh) policy.
Additionally, when running on Universal we have to ensure that every dataplane in the mesh has been configured with a Dataplane Token.

### TrafficPermission

When mTLS is enabled, every connection between dataplanes is denied by default, so we have to explicitly allow it using [TrafficPermission](../../policies/traffic-permissions).

## Postgres

Since on Universal secrets such as `provided` CA's private key are stored in Postgres, a connection between Postgres and Kuma CP should be secured with TLS.

To secure the connection, we first need to pick the security mode using `KUMA_STORE_POSTGRES_TLS_MODE`. There are several modes:

* `disable` - is not secured with TLS (secrets will be transmitted over network in plain text).
* `verifyNone` - the connection is secured but neither hostname, nor by which CA the certificate is signed is checked.
* `verifyCa` - the connection is secured and the certificate presented by the server is verified using the provided CA.
* `verifyFull` - the connection is secured, certificate presented by the server is verified using the provided CA and server hostname must match the one in the certificate.

The CA used to verify the server's certificate can be set using the `KUMA_STORE_POSTGRES_TLS_CA_PATH` environment variable.

After configuring the above security settings in Kuma, we also have to configure Postgres' [`pg_hba.conf`](https://www.postgresql.org/docs/9.1/auth-pg-hba-conf.html) file to restrict unsecured connections.

Here is an example configuration that will allow only TLS connections and will require username and password:
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
hostssl all             all             0.0.0.0/0               password
```

We can also provide a client key and certificate for mTLS using the `KUMA_STORE_POSTGRES_TLS_CERT_PATH` and `KUMA_STORE_POSTGRES_TLS_KEY_PATH` variables. This pair can be used in conjunction with the `cert` auth-method described [here](https://www.postgresql.org/docs/9.1/auth-pg-hba-conf.html).