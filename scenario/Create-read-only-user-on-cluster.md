## Create read only user on all namespace

**Step 1:** create service account
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ahmad
  namespace: default
EOF
```

**Step 2:** create cluster role
api groups: all
resource: all
verb: get, watch, list
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: reader
rules:
- apiGroups: ["*"] # "" indicates the core API group
  resources: ["*"]
  verbs: ["get", "watch", "list"]
EOF
```

**Step 3:** create cluster role binding
cluster role: reader
service account: ahmad
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: reader
subjects:
- kind: ServiceAccount
  name: ahmad
  namespace: default
EOF
```

**Step 4:** get token for ahmad service account
```bash
kubectl -n default create token ahmad
```
