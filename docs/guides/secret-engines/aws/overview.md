---
title: Manage AWS IAM Secrets using the KubeVault operator
menu:
  docs_{{ .version }}:
    identifier: overview-aws
    name: Overview
    parent: aws-secret-engines
    weight: 10
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeVault? Please start [here](/docs/concepts/README.md).

# Manage AWS IAM Secrets using the KubeVault operator

You can easily manage [AWS secret engine](https://www.vaultproject.io/docs/secrets/aws/index.html#aws-secrets-engine) using KubeVault operator.

You should be familiar with the following CRD:

- [AWSRole](/docs/concepts/secret-engine-crds/awsrole.md)
- [AWSAccessKeyRequest](/docs/concepts/secret-engine-crds/awsaccesskeyrequest)
- [AppBinding](/docs/concepts/vault-server-crds/auth-methods/appbinding.md)

Before you begin:

- Install KubeVault operator in your cluster following the steps [here](/docs/setup/operator/install).

- Deploy Vault. It could be in the Kubernetes cluster or external.

To keep things isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace/demo created
```

In this tutorial, we will create [role](https://www.vaultproject.io/api/secret/aws/index.html#create-update-role) using AWSRole and issue credential using AWSAccessKeyRequest. For this tutorial, we are going to deploy Vault using KubeVault operator.

```console
$ cat examples/guides/secret-engins/aws/vault.yaml
apiVersion: kubevault.com/v1alpha1
kind: VaultServer
metadata:
  name: vault
  namespace: demo
spec:
  nodes: 1
  version: "1.0.0"
  backend:
    inmem: {}
  unsealer:
    secretShares: 4
    secretThreshold: 2
    mode:
      kubernetesSecret:
        secretName: vault-keys

$ kubectl get vaultserverversions/1.0.0 -o yaml
apiVersion: catalog.kubevault.com/v1alpha1
kind: VaultServerVersion
metadata:
  name: 1.0.0
spec:
  exporter:
    image: kubevault/vault-exporter:0.1.0
  unsealer:
    image: kubevault/vault-unsealer:0.2.0
  vault:
    image: vault:1.0.0
  version: 1.0.0

$ kubectl apply -f examples/guides/secret-engins/aws/vault.yaml
vaultserver.kubevault.com/vault created

$ kubectl get vaultserver/vault -n demo
NAME      NODES     VERSION   STATUS    AGE
vault     1         1.0.0     Running   1h
```

## AWSRole

Using [AWSRole](/docs/concepts/secret-engine-crds/awsrole.md), you can configure [root IAM credentials](https://www.vaultproject.io/api/secret/aws/index.html#configure-root-iam-credentials) and create [role](https://www.vaultproject.io/api/secret/aws/index.html#create-update-role). In this tutorial, we are going to create `demo-role` in `demo` namespace.

```yaml
apiVersion: engine.kubevault.com/v1alpha1
kind: AWSRole
metadata:
  name: demo-role
  namespace: demo
spec:
  credentialType: iam_user
  policy:
    Version: '2012-10-17'
    Statement:
    - Effect: Allow
      Action: ec2:*
      Resource: "*"
  ref:
    name: vault-app
    namespace: demo
  config:
    credentialSecret: aws-cred
    region: us-east-1
    leaseConfig:
      lease: 1h
      leaseMax: 1h
```

Here, `spec.config.credentialSecret` will be used to configure [root iam credentials](https://www.vaultproject.io/api/secret/aws/index.html#configure-root-iam-credentials).

```yaml
$ cat examples/guides/secret-engins/aws/aws-cred.yaml
apiVersion: v1
data:
  access_key: QUAAAAA=
  secret_key: LKUHGHGJAAAAAAAAAAAAAAAAA==
kind: Secret
metadata:
  name: aws-cred
  namespace: demo
type: Opaque

$ kubectl apply -f examples/guides/secret-engins/aws/aws-cred.yaml
```

`spec.ref` is the reference of AppBinding containing Vault connection and credential information. See [here](/docs/concepts/vault-server-crds/auth-methods/overview.md) for Vault authentication using AppBinding in KubeVault operator.

```yaml
$ cat examples/guides/secret-engins/aws/vault-app.yaml
apiVersion: appcatalog.appscode.com/v1alpha1
kind: AppBinding
metadata:
  name: vault-app
  namespace: demo
spec:
  clientConfig:
    caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN1RENDQWFDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFOTVFzd0NRWURWUVFERXdKallUQWUKRncweE9ERXlNamN3TkRVNU1qVmFGdzB5T0RFeU1qUXdORFU1TWpWYU1BMHhDekFKQmdOVkJBTVRBbU5oTUlJQgpJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBMVhid2wyQ1NNc2VQTU5RRzhMd3dUVWVOCkI1T05oSTlDNzFtdUoyZEZjTTlUc1VDQnlRRk1weUc5dWFvV3J1ZDhtSWpwMVl3MmVIUW5udmoybXRmWGcrWFcKSThCYkJUaUFKMWxMMFE5MlV0a1BLczlXWEt6dTN0SjJUR1hRRDhhbHZhZ0JrR1ViOFJYaUNqK2pnc1p6TDRvQQpNRWszSU9jS0xnMm9ldFZNQ0hwNktpWTBnQkZiUWdJZ1A1TnFwbksrbU02ZTc1ZW5hWEdBK2V1d09FT0YwV0Z2CmxGQmgzSEY5QlBGdTJKbkZQUlpHVDJKajBRR1FNeUxodEY5Tk1pZTdkQnhiTWhRVitvUXp2d1EvaXk1Q2pndXQKeDc3d29HQ2JtM0o4cXRybUg2Tjl6Tlc3WlR0YTdLd05PTmFoSUFEMSsrQm5rc3JvYi9BYWRKT0tMN2dLYndJRApBUUFCb3lNd0lUQU9CZ05WSFE4QkFmOEVCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBTkJna3Foa2lHCjl3MEJBUXNGQUFPQ0FRRUFXeWFsdUt3Wk1COWtZOEU5WkdJcHJkZFQyZnFTd0lEOUQzVjN5anBlaDVCOUZHN1UKSS8wNmpuRVcyaWpESXNHNkFDZzJKOXdyaSttZ2VIa2Y2WFFNWjFwZHRWeDZLVWplWTVnZStzcGdCRTEyR2NPdwpxMUhJb0NrekVBMk5HOGRNRGM4dkQ5WHBQWGwxdW5veWN4Y0VMeFVRSC9PRlc4eHJxNU9vcXVYUkxMMnlKcXNGCmlvM2lJV3EvU09Yajc4MVp6MW5BV1JSNCtSYW1KWjlOcUNjb1Z3b3R6VzI1UWJKWWJ3QzJOSkNENEFwOUtXUjUKU2w2blk3NVMybEdSRENsQkNnN2VRdzcwU25seW5mb3RaTUpKdmFzbStrOWR3U0xtSDh2RDNMMGNGOW5SOENTSgpiTjBiZzczeVlWRHgyY3JRYk0zcko4dUJnY3BsWlRpUy91SXJ2QT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    service:
      name: vault
      port: 8200
      scheme: HTTPS
  parameters:
    serviceAccountName: demo-sa
    policyControllerRole: aws-role
    authPath: kubernetes

$ kubectl apply -f examples/guides/secret-engins/aws/vault-app.yaml
appbinding.appcatalog.appscode.com/vault-app create
```

You need to create `demo-sa` serviceaccount by running following command:

```console
$ kubectl create serviceaccount -n demo demo-sa
serviceaccount/demo-sa created
```

`demo-sa` serviceaccount in the above AppBinding need to have the policy with following capabilities in Vault.

```hcl
path "sys/mounts" {
    capabilities = ["read", "list"]
}

path "sys/mounts/*" {
    capabilities = ["create", "read", "update", "delete"]
}

path "aws/config/root" {
	capabilities = ["create", "read", "update", "delete"]
}

path "aws/config/lease" {
	capabilities = ["create", "read", "update", "delete"]
}

path "aws/roles/*" {
	capabilities = ["create", "update", "read", "delete"]
}

path "aws/creds/*" {
    capabilities = ["read"]
}

path "sys/leases/revoke/*" {
    capabilities = ["update"]
}
```

You can manage policy in Vault using KubeVault operator, see [here](/docs/guides/policy-management/policy-management).

To create policy with above capabilities run following command

```console
$ kubectl apply -f examples/guides/secret-engins/aws/policy.yaml
vaultpolicy.policy.kubevault.com/aws-role-policy created
vaultpolicybinding.policy.kubevault.com/aws-role created
```

Now, we are going to create `demo-role`.

```console
$ cat examples/guides/secret-engins/aws/demo-role.yaml
apiVersion: engine.kubevault.com/v1alpha1
kind: AWSRole
metadata:
  name: demo-role
  namespace: demo
spec:
  credentialType: iam_user
  policy:
    Version: '2012-10-17'
    Statement:
    - Effect: Allow
      Action: ec2:*
      Resource: "*"
  ref:
    name: vault-app
    namespace: demo
  config:
    credentialSecret: aws-cred
    region: us-east-1
    leaseConfig:
      lease: 1h
      leaseMax: 1h

$ kubectl apply -f examples/guides/secret-engins/aws/demo-role.yaml
awsrole.engine.kubevault.com/demo-role created
```

Check whether AWSRole is successful.

```console
$ kubectl get awsroles/demo-role -n demo -o json | jq '.status'
{
  "observedGeneration": "1$6208915667192219204",
  "phase": "Success"
}
```

To resolve the naming conflict, name of the role in Vault will follow this format: `k8s.{spec.clusterName or -}.{spec.namespace}.{spec.name}`.

```console
$ vault list aws/roles
Keys
----
k8s.-.demo.demo-role

$ vault read aws/roles/k8s.-.demo.demo-role
Key                 Value
---                 -----
credential_types    [iam_user]
default_sts_ttl     0s
max_sts_ttl         0s
policy_arns         <nil>
policy_document     {"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"ec2:*","Resource":"*"}]}
role_arns           <nil>
```

If we delete AWSRole, then respective role will be deleted from Vault.

```console
$ kubectl delete -f examples/guides/secret-engins/aws/demo-role.yaml
awsrole.engine.kubevault.com "demo-role" deleted

# check in vault whether role exists
$ vault read aws/roles/k8s.-.demo.demo-role
Error reading aws/roles/k8s.-.demo.demo-role: Error making API request.

URL: GET https://127.0.0.1:8200/v1/aws/roles/k8s.-.demo.demo-role
Code: 400. Errors:

* Role 'k8s.-.demo.demo-role' not found

$ vault list aws/roles
No value found at aws/roles/
```

## AWSAccessKeyRequest

Using [AWSAccessKeyRequest](/docs/concepts/secret-engine-crds/awsaccesskeyrequest), you can issue AWS credential from Vault. In this tutorial, we are going to issue AWS credential by creating `demo-cred` AWSAccessKeyRequest in `demo` namespace.

```yaml
apiVersion: engine.kubevault.com/v1alpha1
kind: AWSAccessKeyRequest
metadata:
  name: demo-cred
  namespace: demo
spec:
  roleRef:
    name: demo-role
    namespace: demo
  subjects:
  - kind: User
    name: nahid
    apiGroup: rbac.authorization.k8s.io
```

Here, `spec.roleRef` is the reference of AWSRole against which credential will be issued. `spec.subjects` is the reference to the object or user identities a role binding applies to and it will have read access of the credential secret. Also, KubeVault operator will use AppBinding reference from AWSRole which is specified in `spec.roleRef`.

Now, we are going to create `demo-cred` AWSAccessKeyRequest.

```console
$ kubectl apply -f examples/guides/secret-engins/aws/demo-cred.yaml
awsaccesskeyrequest.engine.kubevault.com/demo-cred created

$ kubectl get awsaccesskeyrequests -n demo
NAME        AGE
demo-cred   3s
```

AWS credential will not be issued until it is approved. To approve it, you have to add `Approved` in `status.conditions[].type` field. You can use [KubeVault CLI](https://github.com/kubevault/cli) as [kubectl plugin](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/) to approve or deny DatabaseAccessRequest.

```console
# using KubeVault cli as kubectl plugin to approve request
$ kubectl vault approve awsaccesskeyrequest demo-cred -n demo
approved

$ kubectl get awsaccesskeyrequest demo-cred -n demo -o yaml
apiVersion: engine.kubevault.com/v1alpha1
kind: AWSAccessKeyRequest
metadata:
  finalizers:
  - awsaccesskeyrequest.engine.kubevault.com
  name: demo-cred
  namespace: demo
spec:
  roleRef:
    name: demo-role
    namespace: demo
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: nahid
status:
  conditions:
  - type: Approved
```

Once AWSAccessKeyRequest is approved, KubeVault operator will issue credential from Vault and create a secret containing the credential. Also it will create rbac role and rolebinding so that `spec.subjects` can access secret. You can view the information in `status` field.

```console
$ kubectl get awsaccesskeyrequest/demo-cred -n demo -o json | jq '.status'
{
  "conditions": [
    {
      "type": "Approved"
    }
  ],
  "lease": {
    "duration": "1h0m0s",
    "id": "aws/creds/k8s.-.demo.demo-role/1KljT3F5ZrYMC66JsPIgOgXr",
    "renewable": true
  },
  "secret": {
    "name": "demo-cred-wttlrm"
  }
}

$ kubectl get secrets/demo-cred-wttlrm -n demo -o yaml
apiVersion: v1
data:
  access_key: QUtJQUo3TVVIyVFE=
  secret_key: ZFdrNG04Rk05ZkhDbjBlZWNkMk5KUkFBaXB5TkUybw==
  security_token: null
kind: Secret
metadata:
  name: demo-cred-wttlrm
  namespace: demo
type: Opaque
```

If AWSAccessKeyRequest is deleted, then credential lease (if have any) will be revoked.

```console
$ kubectl delete awsaccesskeyrequest/demo-cred -n demo
awsaccesskeyrequest.engine.kubevault.com "demo-cred" deleted
```

If AWSAccessKeyRequest is `Denied`, then KubeVault operator will not issue any credential.

> Note: Once AWSAccessKeyRequest is `Approved` or `Denied`, you can not change `spec.roleRef` and `spec.subjects` field.
