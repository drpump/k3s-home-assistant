# Home assistant in k3s

Home Assistant deployment manifests for k3s with TLS enabled and discovery working via `hostNetwork: true`. 

Manifests should work as-is after changing domain names for certificates and ingress, but they're set up to use `kustomize`. See usage instructions below. 

## Pre-requisites

- you have a domain name
- cert-manager is deployed in k3s
- LetsEncrypt production and staging ClusterIssuers are deployed in k3s
- an endpoint is configured to handle ACME http01 requests to validate domain name ownership
- optionally, dynamic volume provisioning is enabled (i.e. longhorn distributed filesystem deployed and configured as the default storage class)

For more detail regarding cert-manager and letsencrypt config, see https://drpump.github.io/mqtt-k8s-tls/.

## Usage

1. Modify the DNS name in `base/certificate-stg.yaml` to reflect your domain name. Apply using `kubectl apply -f base/certificate-stg.yaml` to test your letsencrypt/acme config. It should create a secret named `home-assistant-stg`. If not, check your cert manager logs and try to resolve any issues, noting that LetEncrypt production servers are rate limited so best to troubleshoot/test against staging. 

1. Edit `overlay/kustomization.yaml` and set:
   - Timezone in the TZ variable (`configMapGenerator`)
   - Your preferred host + domain name in the two `patch` operations
1. Build the deployment by using kubectl's built-in kustomize feature:  
   ```kubectl apply -k overlay/```

## Notes

- Home assistant is deployed using a `StatefulSet` because we want to ensure the volume containing your HA database is retained if you scale to zero or your pod is otherwise restarted/rescheduled. A persistent volume claim on a `Deployment` is deleted by default if you're using dynamically provisioned volumes (e.g. longhorn). The advantage of longhorn with StatefulSet is that the Home Assistant pod can run on any node in your cluster. If you're using the host filesystem then the pod cannot be moved to a different node. 

- The `StatefulSet` is referencing the `stable` docker image tag for home assistant. You might want to override this in the `kustomization.yaml` with a fixed version to avoid surprises if the pod is restarted and you get an "automatic" upgrade. 

- Home assistant is using `hostNetwork: true`, meaning port 8123 is opened on the k3s host node and any discovery broadcasts go to the host network. This is not ideal, but there's a lot of mucking around required to get discovery working otherwise. 

- The `IngressRouteTCP` resource sets up host-based ingress routing on port 443 so that you can hit `https://hass.k3s.example.com` (replace with your domain name) regardless of which node in your cluster is running the Home Assistant pod. The `IngressRouteTCP` resource is specific to Traefik, which is the default ingress/load balancing solution in k3s. You will need to use a different resource to work with nginx or other ingress solutions. 

- If migrating from a standalone Home Assistant instance, you can copy your config directory into the pod filesystem. If you're using longhorn, use `lsblk` in the host OS to find the block devices longhorn has created and mount using `sudo mount /dev/sdX /mnt/hass`. In Ubuntu, the block devices are named `sda`, `sdb` etc. Note that the block device name can change if your pod is rescheduled or restarted. 

- This config was tested in a single node cluster (AMD64, Ubuntu 22.04, k3s v1.27.6). Comments about being able to run the home assistant pod on any node are theoretically correct but not yet tested. 
