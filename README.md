# Lokaler Test
oc apply -k nginx-hub-ingress/overlays/prod

# Oder mit ArgoCD:
# repo: https://dein.repo/git/nginx-hub-ingress
# path: overlays/prod


oc get svc -A -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.spec.type,IP:.status.loadBalancer.ingress[*].ip' | grep LoadBalancer

oc get svc -A -l metallb.universe.tf/address-pool -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,IP:.status.loadBalancer.ingress[*].ip'
