# Authorization Through Open Policy Agent Server

![Auth Flow](https://raw.githubusercontent.com/shpwrck/k8s-security/master/AuthNZ.png)

## [Example](https://github.com/open-policy-agent/contrib/tree/main/k8s_authorization)

### Additional Configuration

[^1]
```
--authorization-mode=Webhook
--authorization-webhook-version=v1
--authorization-webhook-config-file=/etc/kubernetes/api-server/authz-webhook.yaml
--authorization-webhook-cache-authorized-ttl=120s
--authorization-webhook-cache-unauthorized-ttl=30s
```

[^2]

When multiple authorization modules are configured, each is checked in sequence. If any authorizer approves or denies a request, that decision is immediately returned and no other authorizer is consulted. If all modules have no opinion on the request, then the request is denied. A deny returns an HTTP status code 403.

<details>
  <summary>Kubernetes Access Review Request from APIServer to Authorization Server</summary>
  
```json
{
  "apiVersion": "authorization.k8s.io/v1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "team-toucans",
      "verb": "get",
      "resource": "pods",
      "version": "v1"
    },
    "user": "jane",
    "groups": [
      "system:authenticated",
      "devops",
      "team-toucans",
    ]
  }
}
```
  
</details>
<details>
  <summary>Example Rego Policy File</summary>
  
```rego
package k8s.authz

import data.kubernetes
import data.users

# Users in the app-log-readers group should only be allowed reading logs from some pods in a given namespace.
deny[reason] {
    input.spec.resourceAttributes.namespace == "default"
    input.spec.resourceAttributes.verb == "get"
    input.spec.resourceAttributes.resource == "pods/log"

    # Work with groups rather than users directly
    input.spec.groups[_] == "app-log-readers"

    # App log readers can only read logs from pods prefixed with "app"
    not startswith(input.spec.resourceAttributes.name, "app")

    reason := "App log readers may only read logs from pods with app prefix"
}

# Developers should be allowed read access to all namespaces, except for kube-system.
deny[reason] {
    input.spec.resourceAttributes.verb == "get"
    input.spec.groups[_] == "developers"
    input.spec.resourceAttributes.namespace == "kube-system"

    reason := "Developers not allowed read access to namespace kube-system"
}

# Some users should be allowed admin access, but only when being on call.
deny[reason] {
    input.spec.groups[_] == "on-call-admins"
    time.now_ns() < time.parse_rfc3339_ns(data.users[input.user].on_call_start)

    reason := "Admin is not yet on call"
}

# Some users should be allowed admin access, but only when being on call.
deny[reason] {
    input.spec.groups[_] == "on-call-admins"
    time.now_ns() > time.parse_rfc3339_ns(data.users[input.user].on_call_end)

    reason := "Admin is no longer on call"
}

# Developers should be allowed access to any resources with a team name label matching the team the user is in.
deny[reason] {
    input.spec.groups[_] == "developers"

    namespace := input.spec.resourceAttributes.namespace
    resource := input.spec.resourceAttributes.resource
    name := input.spec.resourceAttributes.name
    team := [t | t := groups[_]; startswith(t, "team-")][0]

    not kubernetes[namespace][resource][name].metadata.labels.owner == team
}

decision = {
    "apiVersion": "v1",
    "kind": "SubjectAccessReview",
    "status": {
        "allowed": count(deny) == 0,
        "reason": concat(" | ", deny),
    },
}
```
  
</details>

> An OPA Server can manage policies through disk operations.
> OPA administrative APIs can be disabled.
  
### Implications
  
* Greenfield authorization would have to be done with Rego.
* To my knowledge there is no RBAC to Rego Policy conversion engine. Converting standard RBAC to REGO would be a labor intensive solution.

[^1]: https://blog.styra.com/blog/kubernetes-authorization-webhook
[^2]: https://kubernetes.io/docs/reference/access-authn-authz/authorization/
