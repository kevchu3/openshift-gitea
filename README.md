# Gitea Deployment on OpenShift

The following documentation is based off of instructions for a [Gitea installation on Kubernetes] and adapted for Red Hat OpenShift.

## Installation

Install Helm chart:
```
$ oc create ns gitea
$ oc config set-context --current --namespace=gitea
$ helm repo add gitea-charts https://dl.gitea.io/charts/
$ helm install gitea gitea-charts/gitea -f values.yaml
$ helm repo list
$ helm list

# Customizations on Gitea Helm chart gitea-10.1.3
$ oc adm policy add-scc-to-user nonroot-v2 -z gitea-postgresql
```

Update the Postgresql StatefulSet to address this issue: https://github.com/bitnami/charts/issues/22511
```
$ oc edit statefulset gitea-postgresql

spec.template.spec.containers[0].securityContext.seLinuxOptions: null
```

Apply custom Gitea configurations:
```
$ oc edit secret gitea-inline-config -n gitea
### or ###
$ oc apply -f gitea/gitea-inline-config.secret.yaml
```

Create a route via either method:
```
$ oc create route passthrough gitea-http --hostname=gitea.example.com \
  --service gitea-http --insecure-policy='Redirect' --port='http'
### or ###
$ oc apply -f gitea-http.route.yaml
```

Gitea’s deployments run as a known user and PodSecurityStandards should be set to baseline for the pods to run.
```
$ oc label --overwrite ns gitea pod-security.kubernetes.io/enforce=baseline
$ oc adm policy add-scc-to-user privileged -z default -n gitea

```

Upload certificate to data directory in pod (backed by persistent volume):
```
$ oc rsh gitea-7cc448d79f-72s5p
Defaulted container "gitea" out of: gitea, init-directories (init), init-app-ini (init), configure-gitea (init)
# cd /data/gitea/certs/
# vi cert.pem
# vi key.pem
# chmod 664 cert.pem
# chmod 660 key.pem
# chown git:git cert.pem
# chown git:git key.pem
```

Configure certificates and enable Gitea Actions in `gitea-inline-config` Secret, and restart pod:

Configure Gitea in `gitea.server.config`, convert to base64, edit `gitea-inline-config` Secret, and restart pod.  Also, view app.ini to confirm configuration:
```
$ oc apply -f gitea/gitea-inline-config.secret.yaml
$ oc delete pod gitea-7cc448d79f-72s5p
$ oc rsh gitea-7cc448d79f-4g27b
/ # cat /data/gitea/conf/app.ini
```

Create Gitea admin user:
```
$ oc rsh gitea-7cc448d79f-4g27b
Defaulted container "gitea" out of: gitea, init-directories (init), init-app-ini (init), configure-gitea (init)
# gitea admin user create --admin --username <username> --password <password> --email <email>
```

Optionally, update Gitea Helm chart:
```
helm repo update gitea-charts
helm upgrade gitea gitea-charts/gitea -f values.yaml
helm list
```

## Further Reading

The instructions in this Git repository were used in this published [Red Hat Blog] to stand up Gitea on MicroShift

[Gitea installation on Kubernetes]: https://docs.gitea.io/en-us/install-on-kubernetes/
[Red Hat Blog]: https://www.redhat.com/en/blog/how-to-build-a-home-kubernetes-lab-with-microshift-and-gitops
