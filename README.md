# SOPS with Flux CD: Secrets Management Tutorial

This repository demonstrates how to manage encrypted Kubernetes secrets using **Mozilla SOPS** (Secrets OPerationS) with **GPG/PGP** and **Flux CD** in a GitOps workflow.

---

## Table of Contents

1. [Why SOPS vs. External Secret Stores (OpenBao / Vault)?](#why-sops-vs-external-secret-stores-openbao--vault)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Step-by-Step Tutorial](#step-by-step-tutorial)
   - [Step 1: GPG Key Setup](#step-1-gpg-key-setup)
   - [Step 2: Repository SOPS Configuration (`.sops.yaml`)](#step-2-repository-sops-configuration-sopsyaml)
   - [Step 3: In-Cluster Decryption Setup for Flux](#step-3-in-cluster-decryption-setup-for-flux)
   - [Step 4: Creating and Encrypting Secrets](#step-4-creating-and-encrypting-secrets)
   - [Step 5: Testing & Verifying Reconciliation](#step-5-testing--verifying-reconciliation)
5. [Best Practices & Common Pitfalls](#best-practices--common-pitfalls)
   - [Use `stringData` instead of `data`](#use-stringdata-instead-of-data)
   - [Avoid Double Base64 Encoding](#avoid-double-base64-encoding)
   - [The Flux Self-Management Bootstrap Trap](#the-flux-self-management-bootstrap-trap)

---

## Why SOPS vs. External Secret Stores (OpenBao / Vault)?

When managing secrets in GitOps, there are two primary architectures:
1. **In-Git Encryption (Mozilla SOPS):** Encrypted secret files live directly alongside your manifests in Git and are decrypted in-memory by Flux during reconciliation.
2. **External Secret Stores (OpenBao, HashiCorp Vault, AWS Secrets Manager) + Operator:** Secrets live in an external server, and an operator (like External Secrets Operator - ESO) fetches them into the cluster.

**Why choose SOPS?**
- **Zero Server Infrastructure:** No need to deploy, secure, patch, or maintain a high-availability Vault/OpenBao cluster.
- **Pure GitOps (Single Source of Truth):** Git commits contain both the application code and exact matching encrypted secret versions. Rolling back a Git commit automatically rolls back secret configurations.
- **Native Flux CD Support:** Flux supports SOPS out-of-the-box (PGP/GPG, Age, AWS KMS, Azure Key Vault, GCP KMS) without requiring third-party operators.
- **Ideal for Edge / Dev / Lean Environments:** Lightweight, offline-capable, and simple to set up for small-to-medium infrastructures.

*(Use OpenBao/Vault + ESO instead when you require dynamic secrets with TTL, cross-cluster centralized rotation, or strict compliance audit trails for every secret read access.)*

---

## Architecture Overview

```text
 Developer Machine                         Git Repository (GitHub)                  Kubernetes Cluster (Flux CD)
+-----------------------+                 +-------------------------+             +----------------------------------+
| 1. Plaintext Secret   |                 | Encrypted Secret        |             | kustomize-controller             |
|    (stringData)       |                 | (test-app/              |             |   * reads in-cluster sops-gpg    |
|          │            |                 |  basic-auth.yaml)       |             |   * decrypts Secret in-memory    |
|   sops --encrypt      | ── git push ──> |                         | ── sync ──> |   * creates Secret in cluster    |
|          ▼            |                 |                         |             |          │                       |
| Encrypted Secret      |                 |                         |             |          ▼                       |
| (data is encrypted)   |                 |                         |             | Pod mounts / consumes Secret     |
+-----------------------+                 +-------------------------+             +----------------------------------+
```

---

## Prerequisites

Before starting, ensure you have the following CLI tools installed:

- **[kubectl](https://kubernetes.io/docs/tasks/tools/)** connected to your Kubernetes cluster
- **[flux CLI](https://fluxcd.io/flux/cmd/)** (v2+)
- **[sops CLI](https://github.com/getsops/sops)** (v3.8+)
- **[gpg / GnuPG](https://gnupg.org/)**

---

## Step-by-Step Tutorial

### Step 1: GPG Key Setup

1. **Generate a new GPG key** (if you don't already have one):

   ```bash
   export KEY_NAME="flux-secrets"
   export KEY_COMMENT="Flux SOPS Key"

   gpg --batch --full-generate-key <<EOF
   %no-protection
   Key-Type: RSA
   Key-Length: 4096
   Subkey-Type: RSA
   Subkey-Length: 4096
   Name-Real: ${KEY_NAME}
   Name-Comment: ${KEY_COMMENT}
   Expire-Date: 0
   EOF
   ```

2. **Retrieve the Key Fingerprint:**

   ```bash
   export KEY_FP=$(gpg --list-secret-keys --with-colons "${KEY_NAME}" | grep '^fpr:' | head -n 1 | cut -d: -f10)
   echo "GPG Key Fingerprint: ${KEY_FP}"
   ```

3. **Export the Public Key to the repository:**

   The public key can safely be committed to Git so team members can encrypt secrets:

   ```bash
   gpg --armor --export "${KEY_FP}" > flux-system/.sops.pub.asc
   ```

---

### Step 2: Repository SOPS Configuration (`.sops.yaml`)

Define a `.sops.yaml` configuration at the repository root (or in `flux-system/`):

```yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    pgp: CDDAD9230C22E207B2AB2BD183E0D8C9439B21A2
```

> **Note:** Replace the `pgp:` value with your actual GPG key fingerprint (`${KEY_FP}`).
> `encrypted_regex: ^(data|stringData)$` ensures only secret values are encrypted, keeping metadata (names, namespaces, labels) readable.

---

### Step 3: In-Cluster Decryption Setup for Flux

Flux's `kustomize-controller` needs the private GPG key to decrypt secrets inside the cluster.

1. **Export the private key into Kubernetes as a Secret in namespace `flux-system`:**

   ```bash
   gpg --export-secret-keys --armor "${KEY_FP}" | \
     kubectl create secret generic sops-gpg \
     --namespace=flux-system \
     --from-file=sops.asc=/dev/stdin
   ```

2. **Configure Decryption on your Flux `Kustomization`:**

   In `flux-system/gotk-sync.yaml` (or your application's Flux Kustomization manifest), add `spec.decryption`:

   ```yaml
   apiVersion: kustomize.toolkit.fluxcd.io/v1
   kind: Kustomization
   metadata:
     name: flux-system
     namespace: flux-system
   spec:
     decryption:
       provider: sops
       secretRef:
         name: sops-gpg
     interval: 10m0s
     path: ./
     prune: true
     sourceRef:
       kind: GitRepository
       name: flux-system
   ```

3. **Apply the configuration to the cluster:**

   ```bash
   kubectl apply -f flux-system/gotk-sync.yaml
   ```

---

### Step 4: Creating and Encrypting Secrets

Always use **`stringData:`** for plaintext secrets when working with SOPS.

1. **Create an unencrypted secret manifest** (`test-app/basic-auth.yaml`):

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: basic-auth
     namespace: default
   stringData:
     user: admin
     password: supersecretpassword123
     name: test-app-user
   ```

2. **Encrypt the file with SOPS:**

   ```bash
   sops --encrypt --in-place test-app/basic-auth.yaml
   ```

   *(SOPS automatically uses the rule in `.sops.yaml` to encrypt all fields matching `stringData`)*

3. **Verify the encrypted file:**

   The file should now contain `ENC[AES256_GCM,...]` under `stringData:` and a `sops:` metadata section at the bottom.

4. **Commit and push to Git:**

   ```bash
   git add test-app/basic-auth.yaml
   git commit -m "feat: add encrypted basic-auth secret"
   git push
   ```

---

### Step 5: Testing & Verifying Reconciliation

1. **Trigger Flux reconciliation:**

   ```bash
   flux reconcile kustomization flux-system --with-source
   ```

2. **Check the Kustomization status:**

   ```bash
   flux get kustomizations
   ```
   *Output should show `READY: True`.*

3. **Verify the decrypted Secret in Kubernetes:**

   ```bash
   kubectl get secret basic-auth -n default -o yaml
   ```

4. **Verify that a Pod consumes the decrypted secret:**

   In `test-app/pod.yaml`, a test pod mounts the secret and injects environment variables:

   ```bash
   kubectl get pod basic-auth-app -n default
   ```

   - **Check environment variables:**
     ```bash
     kubectl exec -it basic-auth-app -n default -- env | grep BASIC_AUTH
     ```

   - **Check mounted secret files (in plaintext):**
     ```bash
     kubectl exec -it basic-auth-app -n default -- sh -c 'for f in /etc/secrets/*; do echo -n "$f: "; cat "$f"; echo ""; done'
     ```

---

## Best Practices & Common Pitfalls

### 1. Use `stringData` instead of `data`
- **`data:`** requires base64-encoded values. If you write plaintext under `data:` and encrypt it with SOPS, Kubernetes will fail dry-run validation with `illegal base64 data at input byte ...`.
- **`stringData:`** accepts human-readable plaintext strings and Kubernetes automatically base64-encodes them upon storage.

### 2. Avoid Double Base64 Encoding
- Never pre-encode values with `base64` before putting them into `stringData:`.
- If you base64-encode a value manually and put it into `stringData:`, Kubernetes will base64-encode it a second time. When mounted into the container, the file will still contain base64 instead of the actual plaintext secret.

### 3. The Flux Self-Management Bootstrap Trap
- When running `flux bootstrap`, Flux regenerates `flux-system/gotk-sync.yaml` and removes custom modifications like `spec.decryption`.
- If this happens:
  1. Re-add `spec.decryption` to `flux-system/gotk-sync.yaml`.
  2. Apply it directly with `kubectl apply -f flux-system/gotk-sync.yaml` so the controller has the decryption configuration before it attempts to process encrypted secrets.
  3. Recreate `sops-gpg` secret if the `flux-system` namespace was recreated.
