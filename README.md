# smcp-customize

**First time update GitHub**
```
echo "# smcp-customize" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/smcp-customize.git
git config --global credential.helper 'cache --timeout 7200'
git push -u origin main


git add . ; git commit -a -m "update README" ; git push -u origin main
```

## Deploying extra ingress/egress gateways using ServiceMeshControlPlane

Template for SMCP 
- https://docs.openshift.com/container-platform/4.10/service_mesh/v2x/ossm-reference-smcp.html

Deploying extra ingress/egress gateways using ServiceMeshControlPlane resource Reference: 
- https://access.redhat.com/articles/6619501

Configure custom certificate for the Service Mesh ingress gateway in RHOCP 4 Reference:
- https://access.redhat.com/solutions/6650301
