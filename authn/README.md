# Authentication Through OAuth2.0 (Ping/Dex/Okta/etc)

![Authentication Flow](https://raw.githubusercontent.com/shpwrck/k8s-security/master/AuthNZ.png)

## [Example](https://dexidp.io/docs/kubernetes/)

### Additional Configuration

__API Server__ 
```
--oidc-issuer-url=https://dex.example.com:32000
--oidc-client-id=example-app
--oidc-ca-file=/etc/ssl/certs/openid-ca.pem
--oidc-username-claim=email
--oidc-groups-claim=groups
```

> *The APIServer does not need to communicate with the Identity Provider during the authentication flow*

The above will require either:

* `kubeconfig` to include
```
users:
- name: (USERNAME)
  user:
    token: (ID-TOKEN)
```
* `kubectl` to include
```
--token='': Bearer token for authentication to the API server
```

### Implications

[^1]
Since all of the data needed to validate who you are is in the id_token, Kubernetes doesn't need to "phone home" to the identity provider. In a model where every request is stateless this provides a very scalable solution for authentication. It does offer a few challenges:

Kubernetes has no "web interface" to trigger the authentication process. There is no browser or interface to collect credentials which is why you need to authenticate to your identity provider first.
The id_token can't be revoked, it's like a certificate so it should be short-lived (only a few minutes) so it can be very annoying to have to get a new token every few minutes.
To authenticate to the Kubernetes dashboard, you must use the kubectl proxy command or a reverse proxy that injects the id_token.

[^1]: https://kubernetes.io/docs/reference/access-authn-authz/authentication/
