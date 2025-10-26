#  Sealed Secret Decryption Issue Documentation

## Argocd app shows 'degraded' though the key unsealed successfully and sealed secret health status shows wrong

##  Issue Summary
While applying a new **SealedSecret** file, the following error occurred:

```
no key could decrypt secret 
```
<img width="1527" height="370" alt="kubeseal error" src="https://github.com/user-attachments/assets/0d5b4634-be9a-4d42-b3f1-9a0e1b987fbf" />

------

<img width="910" height="353" alt="image" src="https://github.com/user-attachments/assets/c8364207-18ce-403e-a3c3-4dabba1d136b" />


This happens when the secret is sealed with an outdated or mismatched certificate, or when it’s sealed without specifying **cluster-wide scope**.

---

##  Root Cause
The **Sealed Secrets controller** uses a private key to decrypt secrets encrypted using its public key.  
If the secret is sealed with a stale public certificate or the wrong scope, it cannot be decrypted successfully.

---

##  Solution Steps

### 1. Re-seal the secret using cluster-wide scope
Use the following command to seal the secret correctly:

```bash
kubeseal --format=yaml   --cert=pub-sealed-secrets.pem   --scope cluster-wide   < /tmp/dev-mongodb-secret.yaml   > mongodb-sealedsecret.yaml
```

**Explanation:**
- `--format=yaml` → Output in YAML format.  
- `--cert` → Path to the controller’s public certificate.  
- `--scope cluster-wide` → Ensures the secret can be decrypted across namespaces.

---

### 2. Verify the current certificate
If you’re not sure whether your certificate is up to date, fetch the latest one from the cluster:

```bash
kubeseal --fetch-cert   --controller-namespace kube-system   --controller-name sealed-secrets-controller   > cluster-cert.pem
```

Then re-run the sealing command using `cluster-cert.pem`.

---

## Example Workflow

```bash
# Step 1: Create the Kubernetes Secret manifest
kubectl create secret generic mcp-mongodb   --from-literal=MONGODB_URI="your-mongodb-uri"   --dry-run=client -o yaml > /tmp/dev-mongodb-secret.yaml

# Step 2: Fetch the current Sealed Secrets public key
kubeseal --fetch-cert   --controller-namespace kube-system   --controller-name sealed-secrets-controller   > cluster-cert.pem

# Step 3: Seal the secret (cluster-wide)
kubeseal --format=yaml   --cert=cluster-cert.pem   --scope cluster-wide   < /tmp/dev-mongodb-secret.yaml   > mcp-mongodb-sealedsecret.yaml

# Step 4: Apply the sealed secret
kubectl apply -f shopbrain-mcp-mongodb-sealedsecret.yaml
```
## Or copy the file and paste it into your repo if you are using ArgoCD and click sync
---

<img width="1512" height="608" alt="kube seal resolved" src="https://github.com/user-attachments/assets/f7d2c046-07b5-4f26-9ba1-61e8af6d5bf1" />

##  Tips
- Always use `--scope cluster-wide` unless your controller requires namespace scope.  
- Re-fetch the certificate if the controller was redeployed or updated.  
- Keep a copy of the public certificate in your repo for consistent team use.
