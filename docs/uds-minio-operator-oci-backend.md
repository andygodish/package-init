# Changing from uds-dev-stack MinIO Backend to uds-minio-operator MinIO Backend

This became more of a challenge than initially expected. Using `configs/minio-backend-2.toml`, I redeployed the init package to point to a different MinIO tenant derived from the UDS-integrated MinIO Operator. The connection strings in that configuration file worked as expected, however, the ztunnel logs indicated an error between the docker-registry pods and the minio downstream service.

There was a discrepancy here that I still do not fully understand. The ztunnel logs error indicated this destination: `dst.service="minio.minio.svc.cluster.local"`. This was not expected as the zarf config utilizes the `uds-minio-hl` service's DNS name. There exists a `minio` service in that namespace, but it serves port 80 and not 9000 as used by the connection string. The following error messages were also found in the ztunnel logs related to the connection between the docker-registry and minio services:

```txt
connection closed due to policy rejection: explicitly denied by: istio-system/istio_converted_static_strict
```

I created a peerauthentication resource,

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: minio-debug
  namespace: minio
spec:
  mtls:
    mode: PERMISSIVE
```

and the error changed to this:

```txt
connection closed due to policy rejection: allow policies exist, but none allowed
```

I then created a more permissive authorizationpolicy resource and the connections were then allowed through:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-all-zarf-to-minio-9000-debug
  namespace: minio
spec:
  action: ALLOW
  rules:
  - to:
    - operation:
        ports: ["9000"]
```
