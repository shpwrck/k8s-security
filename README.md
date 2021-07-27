# Kubernetes Security Concerns

## Motivations

* Administration of Kubernetes cannot be done with Kubernetes
* Control traffic and workload traffic cannot share the same network interface

## General Concepts

* To provide additional security to the Kubernetes control plane a few nonstandard components can be installed:
  * external authentication server
  * external authorization server
  * egress proxy/firewall for the apiserver
* In order for the above to provide value, authN and authZ must be required for all requests.
* For further security the external AuthN/AuthZ should restrict access to any of the management apis.
  * For instance, webhook configuration apis, podsecuritypolicy apis, networkpolicy apis.
* Additional security implications exist surrounding CSI, CNI, and CloudProviders that have been omitted from this document.

## Kubernetes Component + Relevant Configuration

### kubelet

The kubelet can receives pod manifests from the api-server and reports status back.
It can be configured to run in a disconnected mode if desired.

Recommended Congfiguration:

* Disable anonymous authentication
* Switch authorization to Webhook mode
* Disable Alpha/Beta APIs
* Enable Kube/System Reservation (Prevent DOS)
* Disable `read-only-port` (require authn/z)
* Ensure that the Kubelet only makes use of Strong Cryptographic Ciphers

Additional Configuration:

* Disable `--enable-controller-attach-detach` if there is a concern CSI vulnerabilities
* Modify cluster-dns/resolv-conf settings if CoreDNS is untrusted

### kube-apiserver

The APIServer communicates all changes to Kubernetes resources to the datastore.

Recommended Configuration:

* Explicitly Identify bind-address
* Disallow anonymous-auth
* Ensure that the --kubelet-https argument is set to true
* Ensure that the --authorization-mode argument is not set to AlwaysAllow
* Ensure that the API Server only makes use of Strong Cryptographic Ciphers
* Enable auditing and the various rates/boundaries therein
* `authentication-token-webhook-config-file` will enable off-cluster identity
* `authorization-policy-file/authorization-webhook-config-file` will enable off-cluster authority
* Use `disable-admission-plugins` to disable Mutating and Validating Admission Webhooks
* Disable Alpha/Beta APIs `--runtime-config` and `--feature-gates`
* Use `--tls-min-version` to set the minimum TLS Version

Additional Configuration:

* Service account maintenance (OIDC) as necessary
* Allow/disallow privileged containers as required
* `--egress-selector-config-file` will allow you to inject an additional network security device between the api-server and the cluster

### kube-controller-manager

Watches the api-server and implements changes on other components.

Recommended Configuration:

* `--authorization-always-allow-paths` set to '' to require authorization on all paths
* `--feature-gates` again to remove alpha/beta
* Use `--tls-min-version` to set the minimum TLS Version
* Ensure that the API Server only makes use of Strong Cryptographic Ciphers

Additional Configuration:

* `--controllers` to enable or disable on a per-controller basis
* `use-service-account-credentials` to eliminate the reuse of credentials for all controllers

### kube-proxy

Handles pod and service IP delegation as well as loadbalancing.

Recommended Configuration:

* `--feature-gates` again to remove alpha/beta

Additional Configuration:

* N/A

### kube-scheduler

Determines node level resources through the api-server delegates to specific kubelets.

Recommended Configuration:

* `--bind-address` specify otherwise "all interfaces will be used"
* `--feature-gates` again to remove alpha/beta
* Use `--tls-min-version` to set the minimum TLS Version
* Ensure that the API Server only makes use of Strong Cryptographic Ciphers

Additional Configuration:

* N/A
