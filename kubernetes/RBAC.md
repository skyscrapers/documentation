# Role Based Access Control

All clusters are configured with Role Based Access Control (RBAC).
The authentication controller checks each API call with its attached role to allow or deny the call.

## Enabling RBAC

RBAC is enabled by default on each cluster.
You can check it by looking at the `apiserver` flags:
`kubectl -n kube-system get pods -l k8s-app=kube-apiserver -o=jsonpath='{.items[0].spec.containers[0].command}' | grep -o authorization-mode="\w"\*`

## Granting permissions

Each process or person making API calls must provide authentication (a token) and have permissions to make that call. You can grant permissions by applying the `Role`, `RoleBinding` and optionally `ServiceAccount` objects to the cluster. Our default roles are created automatically with the [`kubesignin` chart](https://github.com/skyscrapers/charts/tree/master/kubesignin).

In order to create custom roles you can add them in a chart or apply them manually creating a `role.yaml` file and apply it to the cluster with the command: `kubectl apply -f role.yaml`.

NOTE: When using GitHub authentication, any changes in the GitHub teams will require you to create a new token to reflect the new permissions. This won't be needed if you change permissions in the k8s cluster.

## ServiceAccount

`ServiceAccounts` are special accounts for processes making API calls. These are not intended for human use.

You can specify which `ServiceAccount` to use in the Pod spec.

### Roles and ClusterRoles

Sets of permissions are called `Roles` or `ClusterRoles`.
The difference between the two is the scope. `Roles` are scoped to a namespaces, `ClusterRoles` apply to the full cluster.

There are also predefined `Roles` that Kubernetes provides by default.

`Roles` can be quite extensive. Check the official docs how to write them: <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>.

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
apiVersion: rbac.authorization.k8s.io/v1beta1
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
apiVersion: rbac.authorization.k8s.io/v1beta1
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
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: readonly-staging
  namespace: staging
rules:
  - apiGroups:
      - ""
    resources: ["*"]
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1beta1
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
