# hataraki_cluster

## Setup Instructions

Populate `infra/ansible/inventories/prod/hosts.yaml` file, then roll out cluster with ansible

```bash
cd infra/ansible
ansible-playbook playbooks/site.yaml
```

## Keycloak configuration

- Keycloak first user has admin, so the first login another admin should be created and the temporal admin should be deleted.
- In google auth https://console.cloud.google.com/auth/clients, a Oauth client must be created, capturing `client-id` and `client secret`.
- In keycloak an Identity Provider with Google must be created.
- Use the `client-id` and `client secret` from google and set them in the Identy Provider configuration, then capture `Redirect URI` and store it in Oauth in google console.
- Create a realm: hatarakiassistant
- Create a client called `vault` with the following inputs:
* Client-id: vault
* Valid redirect URIs: <VAULT_HOSTNAME>/ui/vault/auth/oidc/oidc/callback
* Client Authentication: true
* PKCE Method: S256
* Create a role: vault-role
* Capture token`: Client Secret

## Vault Configuration

- Setup the first secret engine.
- Setup a new authentication method, this must be done via CLI since UI does not support all options to enable keycloak OAUTH
- On vault CLI:
```bash
kubectl exec -it $VAULT_POD_NAME -n vault -- sh
export VAULT_TOKEN=$VAULT_ROOT_TOKEN
# Configure oidc
vault write auth/oidc/config oidc_discovery_url="$KEY_CLOAK_URL/realms/hatarakiassistant" \
   oidc_client_id='vault' \
   oidc_client_secret=$CLIENT_SECRET \
   default_role='vault-role'

vault policy write admin-policy - <<EOF
path "$SECRET_ENGINE_NAME/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF

vault write auth/oidc/role/vault-role \
    bound_audiences="vault" \
    allowed_redirect_uris="https://$VAULT_HOSTNAME/ui/vault/auth/oidc/oidc/callback" \
    user_claim="email" \
    policies="default,admin-policy" \
    ttl="1h"
```

## Access Management Vault

After the first user has login with a google account, a new entity will appear, mapping the email to the name.
Go to Access - Groups, create a new group, add the existing policy `admin-policy`, and find the authenticated entity via google to then add it.
Once the user logout and login, he/she should be able to see the desired secret engine.

Create an `kube` policy with the following permissions:
```hcl
path "hatarakiassistant_secrets/*" {
  capabilities = ["read"]
}
```

Enable kubernetes auth in vault.



## Notes

- Flux will automatically sync every 10 minutes
- Dependencies ensure components deploy in correct order
- cert-manager will automatically provision SSL certificates
- Traefik will use the wildcard certificate for HTTPS
- All apps will automatically get certificates through the gateway