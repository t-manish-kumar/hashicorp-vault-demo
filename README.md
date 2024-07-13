# hashicorp-kv-demo

Create the vault namespace:

sh

kubectl create namespace vault

Install HashiCorp Vault using Helm:

sh

helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --set="server.dev.enabled=true" \
  --set="ui.enabled=true" \
  --set="ui.serviceType=LoadBalancer" \
  --namespace vault

Create a policy for reading secrets (read-policy.hcl):

sh

cat <<EOF > /home/vault/read-policy.hcl
path "secret*" {
  capabilities = ["read"]
}
EOF

Write the policy to Vault:

sh

vault policy write read-policy /home/vault/read-policy.hcl

Enable Kubernetes authentication in Vault:

sh

vault auth enable kubernetes

Configure Kubernetes authentication in Vault:

sh

vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://${KUBERNETES_PORT_443_TCP_ADDR}:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

Create a Kubernetes service account (vault-serviceaccount) and bind it to the vault namespace:

yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-serviceaccount
  namespace: vault
  labels:
    app: read-vault-secret

Apply this with:

sh

kubectl apply -f serviceaccount.yaml -n vault

Create a Deployment (vault-test) that uses the vault-serviceaccount to access secrets from Vault:

yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-test
  namespace: vault
  labels:
    app: read-vault-secret
spec:
  replicas: 1
  selector:
    matchLabels:
      app: read-vault-secret
  template:
    metadata:
      labels:
        app: read-vault-secret
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/agent-inject-secret-login: "secret/login"
        vault.hashicorp.com/agent-inject-template-login: |
          {{- with secret "secret/login" -}}
          pattoken={{ .Data.data.pattoken }}
          {{- end }}
        vault.hashicorp.com/agent-inject-secret-my-first-secret: "secret/my-first-secret"
        vault.hashicorp.com/agent-inject-template-my-first-secret: |
          {{- with secret "secret/my-first-secret" -}}
          username={{ .Data.data.username }}
          password={{ .Data.data.password }}
          {{- end }}
        vault.hashicorp.com/role: "vault-role"
    spec:
      serviceAccountName: vault-serviceaccount
      containers:
        - name: nginx
          image: nginx

Apply this with:
kubectl apply -f deployment.yaml -n vault
