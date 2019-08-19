# Role Based Access Control

All clusters are configured with Role Based Access Control (RBAC). The authentication controller checks each API call with its attached role to allow or deny the call.

On this page you can find some generic documentation and examples on RBAC. Please make sure to [consult the official documentation too](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

## Enabling RBAC

RBAC is enabled by default on each cluster. You can check it by looking at the `apiserver` flags:
`kubectl -n kube-system get pods -l k8s-app=kube-apiserver -o=jsonpath='{.items[0].spec.containers[0].command}' | grep -o authorization-mode="\w"\*`

## Granting permissions

Each process or person making API calls must provide authentication (a token) and have permissions to make that call. You can grant permission by applying the `Role`, `RoleBinding` and optionally `ServiceAccount` objects to the cluster. By default we provide full admin access (`cluster-admin` `ClusterRole`) for your team:

- For **kops-based clusters** by setting the `addons.kubesignin_k8s_admins_groups` variable in your cluster definition file to the GitHub team of choice.
- For **EKS-based clusters** by setting the `cluster.aws_auth_admin_role` and/or `cluster.aws_auth_role_mappings` variables in your cluster definition file to the AWS IAM role(s) of choice.

In order to create custom `Roles` you can add them in a chart or apply them manually by creating a `role.yaml` file and apply it to the cluster with the command: `kubectl apply -f role.yaml`.

NOTE: When using GitHub authentication, any changes in the GitHub teams will require you to create a new token to reflect the new permissions. This won't be needed if you change permissions in the K8s cluster.

## ServiceAccount

`ServiceAccounts` are special accounts for processes making API calls. These are not intended for human use.

You can specify which `ServiceAccount` to use in the Pod spec.

### Roles and ClusterRoles

Sets of permissions are called `Roles` or `ClusterRoles`.
The difference between the two is the scope. `Roles` are scoped to a `Namespace`, `ClusterRoles` apply to the full cluster.

There are also predefined `Roles` that Kubernetes provides by default.

`Roles` can be quite extensive. Be sure to check the official docs how to write them: <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>.

**IMPORTANT**: Due to a serious vulnerability, which hasn't been patched for K8s 1.11 and K8s 1.12, it's possible for users to allow acces to cluster-wide CRDs as if it were namespaced. This means when you should not grant access in your namespaced Roles to grant access on `resources: [*], apiGroups: [*]`! Always follow the best practice of explicitely granting the least amount of privileges. See the [CVE-2019-11247 issue](https://github.com/kubernetes/kubernetes/issues/80983) and [RBAC documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) for more details.

### (Cluster)RoleBinding

A `RoleBinding` is what binds a `ServiceAccount` or group to a `Role`.

### Examples

Create a `ServiceAccount`, `Role` and `RoleBinding`:

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app: MyApp
  name: MyApp-account
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app: MyApp
  name: MyApp-role
rules:
  - apiGroups:
      - ""
    resources:
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - update
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app: MyApp
  name: MyApp-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: MyApp-role
subjects:
  - kind: ServiceAccount
    name: MyApp-account
    namespace: staging
```

Create a read-only `RoleBinding` for users from the `foo` GitHub organisation in the `bar` team:

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: readonly-staging
  namespace: staging
rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-staging-binding
  namespace: staging
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: readonly-staging
subjects:
  - kind: Group
    name: foo:bar
```
