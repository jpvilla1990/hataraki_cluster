# hataraki_cluster

## Flux Setup Instructions

5. **Bootstrap Flux** (from your Ansible or manually):
   ```bash
   flux bootstrap github \
     --owner=<your-github-username> \
     --repository=k8s-gitops \
     --branch=main \
     --path=clusters/production \
     --personal
   ```

6. **Verify deployment:**
   ```bash
   flux get kustomizations
   kubectl get helmreleases -A
   kubectl get certificates -A
   kubectl get gateway -n traefik
   ```

## Notes

- Flux will automatically sync every 10 minutes
- Dependencies ensure components deploy in correct order
- cert-manager will automatically provision SSL certificates
- Traefik will use the wildcard certificate for HTTPS
- All apps will automatically get certificates through the gateway

## Adding New Apps

Just create a new directory under `apps/base/` with:
- deployment.yaml
- service.yaml
- httproute.yaml
- kustomization.yaml

Flux will automatically deploy it!