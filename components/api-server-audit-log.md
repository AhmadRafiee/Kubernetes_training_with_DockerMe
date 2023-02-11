## Kubernetes auditing config

**Step 1:** Connect to control-plane
Connect to the control-plane node and create a directory to host the audit policy as well as audit logs.
```bash
mkdir /etc/kubernetes/audit
```

**Step 2:** Create an audit policy
Create an audit policy file named /etc/kubernetes/audit/policy.yaml with the following data:
```bash
cat <<EOF > /etc/kubernetes/audit/policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:

- level: None
  verbs: ["get", "watch", "list"]

- level: None
  resources:
  - group: "" # core
    resources: ["events"]

- level: None
  users:
  - "system:kube-scheduler"
  - "system:kube-proxy"
  - "system:apiserver"
  - "system:kube-controller-manager"
  - "system:serviceaccount:gatekeeper-system:gatekeeper-admin"

- level: None
  userGroups: ["system:nodes"]

- level: RequestResponse
EOF
```

**Step 3:** Add required entries
Add following entries in /etc/kubernetes/manifests/kube-apiserver.yaml:

Add these line under command section:
```bash
- --audit-policy-file=/etc/kubernetes/audit/policy.yaml
- --audit-log-path=/etc/kubernetes/audit/audit.log
- --audit-log-maxsize=500
- --audit-log-maxbackup=3
```

Now mount this volume by adding the following data under the volumeMounts section:
```bash
- mountPath: /etc/kubernetes/audit
  name: audit
```

Scroll down and add a volume under the volumes section:
```bash
- hostPath:
    path: /etc/kubernetes/audit
    type: DirectoryOrCreate
  name: audit
```

**Step 4:** check audit log
```bash
tail -f /etc/kubernetes/audit/audit.log | jq
```
[Reference](https://signoz.io/blog/kubernetes-audit-logs/)