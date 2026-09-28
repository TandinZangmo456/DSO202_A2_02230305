# DSO202 Assignment 2: Task Tracker on Kubernetes

**Cluster:** kind, cluster name `dso202-a2`
**Namespace:** `dso202-assignment-02`
**URL:** `https://tasktracker.localtest.me`

This assignment extends the three-tier Task Tracker from Assignment 1 (frontend, backend, PostgreSQL). It adds TLS at the Ingress, a StatefulSet for the database, RBAC with a certificate user, and a Custom Resource Definition.

> Fill in the `TODO` markers before submitting. Screenshot paths assume the images are saved under `docs/screenshots/`.

Private keys, certificates and kubeconfigs are **not** in the repository. They live outside the project (`~/dso202-certs` and the TLS key folder), and `git ls-files | grep -E '\.(key|crt|kubeconfig)$'` returns nothing.

## 1. Deploying from scratch

```bash
kubectl apply -f namespace.yaml
kubectl apply -f config/                  # ConfigMap and Secret
kubectl apply -f rbac/serviceaccounts.yaml
kubectl apply -f database/statefulset.yaml
kubectl apply -f backend/deployment.yaml
kubectl apply -f frontend/deployment.yaml
# create the TLS secret from the self-signed cert (see section 3), then:
kubectl apply -f ingress/
kubectl apply -f crd/tasklist-crd.yaml
kubectl apply -f crd/tasklist-cr.yaml
```

The Service manifests, including the headless `db-svc`, are applied with their tier. The database Service must exist before the StatefulSet, because `serviceName: db-svc` refers to it. TODO: confirm the exact apply order against your repo.

Check that everything is healthy:

```bash
kubectl get pods,pvc,svc,ingress
curl -sk https://tasktracker.localtest.me/api/status
# {"status":"ok","db":"connected"}
```

## 3. Ingress and TLS termination

The Ingress routes by host (`tasktracker.localtest.me`) and path (the frontend at `/`, the API under `/api`). TLS terminates at the Ingress using a self-signed certificate stored in the Secret `tasktracker-tls`. Traffic inside the cluster stays plain HTTP.

- The certificate is `CN=tasktracker.localtest.me, O=dso202`.
- `kubectl describe ingress` reports that `tasktracker-tls` terminates `tasktracker.localtest.me`.
- The API answers over HTTP/2 with `db: connected`.
- The browser shows the page over `https://` with all four tasks.

**Screenshot 7:** Ingress description, served certificate, API status

**Screenshot 8:** page loaded over HTTPS

**"Not secure" warning.** The browser shows "Not secure" because the certificate is self-signed and no public certificate authority vouches for it. The connection is still encrypted, but the browser cannot verify who issued the certificate. In production this would be replaced with a certificate from a trusted CA (for example Let's Encrypt via cert-manager).

## 4. StatefulSet for the database (section 2.1)

In Assignment 1 the database was a Deployment plus a standalone PVC. Both are replaced by one StatefulSet (`database/statefulset.yaml`).

Key points in the manifest:

- `serviceName: db-svc` points at the existing **headless** Service (`clusterIP: None`), which gives each Pod a stable DNS name.
- `volumeClaimTemplates` generates one PVC per replica, named `data-db-<ordinal>`. The Pod `db-0` gets `data-db-0`.
- `replicas: 1`. Kubernetes gives replicas an identity and their own storage, but it does not set up database replication. Three replicas would be three unrelated databases, so the database stays at one.
- `persistentVolumeClaimRetentionPolicy` is `Retain` for both `whenDeleted` and `whenScaled`, so data survives deleting or scaling down the StatefulSet.
- The old `deployment.yaml` and `pvc.yaml` were removed from the repository so the files match the cluster.

### Deployment vs StatefulSet

| | Assignment 1 (Deployment + PVC) | Assignment 2 (StatefulSet) |
|---|---|---|
| Pod name after deletion | new random suffix | always `db-0` |
| Storage | standalone PVC referenced by name | `data-db-0`, generated per replica |
| DNS | one Service name only | `db-0.db-svc...` per Pod, plus `db-svc` |
| Startup and update order | none | ordered (`OrderedReady`) |

### Evidence

**Screenshot 9:** `db-0` running, `data-db-0` Bound, `db: connected`

**Screenshot 10:** stable identity and per-Pod DNS

```bash
kubectl get pod db-0 -o jsonpath='{.spec.hostname}.{.spec.subdomain}{"\n"}'   # db-0.db-svc
kubectl exec deploy/backend -- nslookup db-0.db-svc.dso202-assignment-02.svc.cluster.local   # 10.244.0.18
kubectl exec deploy/backend -- nslookup db-svc                                 # 10.244.0.18
```

Both names return the same IP because only one Pod is ready behind the headless Service. With more replicas, `db-svc` would return all of their IPs, while `db-0.db-svc...` would still point only at `db-0`.

The backend image uses BusyBox `nslookup`. It does not try the search domains for a name that already contains a dot, so the short `db-0.db-svc` returns `NXDOMAIN`. The fully qualified name resolves correctly. The `NXDOMAIN` lines and exit code 1 on the `db-svc` lookup come from BusyBox trying every search suffix. The real name still resolves.

**Screenshot 11:** the Pod is deleted, recreated with the same name, and the same PVC and data are back

After `kubectl delete pod db-0`, the new Pod is again `db-0`. `data-db-0` stayed `Bound` to the same volume (`pvc-83fa0272-...`) and kept its age, so the claim was reused rather than recreated. The task created before the deletion is still in the list.

**Limitation.** After the database Pod restarts, the backend sometimes restarts once (`RESTARTS 1`) before it reconnects. A StatefulSet keeps the database's identity and data, but it does not make the backend tolerate a database restart. Adding retry logic or a readiness probe to the backend would remove this.

## 5. RBAC

### 5.1 ServiceAccount per tier

`rbac/serviceaccounts.yaml` creates `frontend-sa`, `backend-sa` and `db-sa`, and each workload sets `serviceAccountName` to its own. None of the tiers call the Kubernetes API, so each ServiceAccount has `automountServiceAccountToken: false` and no API token is mounted into the containers. That follows least privilege: each tier has its own identity, and a compromised container has no credentials to the API.

**Screenshot 15:** each Pod on its own ServiceAccount, all ready, `db: connected`

TODO: renumber this screenshot if the brief assigns it a different number.

### 5.2 Certificate user

Kubernetes has no user objects. A user is a client certificate signed by the cluster CA: the `CN` becomes the username and the `O` becomes the group.

1. Generate a private key and a CSR with `CN=task-reader` and `O=task-tracker-readers`.
2. Submit a `CertificateSigningRequest` (`rbac/task-reader-csr.yaml`) with the signer `kubernetes.io/kube-apiserver-client` and a 7-day expiry.
3. Approve it with `kubectl certificate approve task-reader`.
4. Extract the issued certificate from `.status.certificate`.

**Screenshot 12:** CSR `Pending` then `Approved,Issued`, and the certificate details

The certificate is issued by `CN=kubernetes` and is valid for 7 days (Sep 28 to Oct 5, 2026).

### 5.3 Read-only Role and RoleBinding

`rbac/task-reader-role.yaml` defines a namespaced Role that allows `get`, `list` and `watch` on Pods, Pod logs, Services, Endpoints, ConfigMaps, PVCs, Deployments, StatefulSets, ReplicaSets and Ingresses. It deliberately **excludes Secrets**. The RoleBinding attaches the Role to the **group** `task-tracker-readers`, which matches the certificate's `O`.

**Screenshot 13:** impersonation tests and the real-certificate tests

| Check | Result |
|---|---|
| list pods | yes |
| get deployments | yes |
| get secrets | no |
| delete pods | no |
| create deployments | no |
| list pods in `kube-system` | no |

With a separate kubeconfig built from the real certificate, `get pods` works, while `get secrets` and `delete pod db-0` return `Forbidden` and name `User "task-reader"`.

**Authentication vs authorisation.** The certificate proves who `task-reader` is, because the cluster CA signed it. That is authentication. The Role and RoleBinding decide what that identity may do. That is authorisation. Before the binding existed, the same certificate would have authenticated but been denied everything.

## 6. Custom Resource Definition

`crd/tasklist-crd.yaml` adds a namespaced `TaskList` type in the group `tasktracker.dso202.io` (version `v1`, short name `tl`). Its `openAPIV3Schema` requires `environment`, `frontendReplicas` and `backendReplicas`, and constrains them:

- `environment` must be one of `dev`, `staging` or `prod`.
- `frontendReplicas` and `backendReplicas` must be integers from 1 to 5.
- `databaseStorage` must match `^[0-9]+(Mi|Gi)$`.
- `host` is a free string.

`additionalPrinterColumns` show Environment, Frontend, Backend and Age in `kubectl get tasklists`. `crd/tasklist-cr.yaml` is a valid instance, `tasktracker-config`, whose values match what is running.

**Screenshot 14:** CRD, valid resource table, and a rejected manifest

An invalid manifest (`environment: production`, `frontendReplicas: 50`, `databaseStorage: lots`, no `backendReplicas`) is rejected by the API server with four validation errors, and nothing is stored.

**Limitation.** There is no operator or controller in this assignment. The CRD stores and validates the object, but nothing watches it. Changing `frontendReplicas` to 3 in the custom resource would **not** scale the frontend. A controller that watches `TaskList` and reconciles the Deployments would be the next step.

## 7. Known limitations

- The TLS certificate is self-signed, so browsers show "Not secure".
- The database runs as a single replica. A StatefulSet gives identity and storage but no PostgreSQL replication.
- The backend may restart once when the database Pod restarts.
- The CRD has no controller behind it.
- The `task-reader` certificate expires after 7 days.
- Secrets are stored as base64 in the cluster with no encryption at rest configured.

## 8. Clean-up

```bash
kubectl delete namespace dso202-assignment-02
kubectl delete csr task-reader
kubectl delete crd tasklists.tasktracker.dso202.io
kind delete cluster --name dso202-a2
```

The CSR and the CRD are cluster-scoped, so deleting the namespace does not remove them.