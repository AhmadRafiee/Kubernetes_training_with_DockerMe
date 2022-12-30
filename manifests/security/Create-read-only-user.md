## Create read only user on mon namespace

**Step a:** create namespace
```bash
kubectl create namespace mon
```

**Step 2:** create service account
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ahmad
  namespace: mon
EOF
```

**Step 2:** create role
api groups: all
resource: all
verb: get, watch, list
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: mon
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
kind: RoleBinding
metadata:
  name: reader
  namespace: mon
roleRef:
  kind: Role
  name: reader
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: ahmad
  namespace: mon
EOF
```

**Step 4:** get token for ahmad service account
```bash
kubectl -n mon create token ahmad
```
