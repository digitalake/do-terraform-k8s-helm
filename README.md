# <h1 align="center">Kubernetes & Helm feat. Terraform</a>

> [!IMPORTANT]
> Visit my Django-demo using [link](https://django.itsyndicate.dns.navy)

### Short description

In this repo You can find:

  - Terraform code for deploying:
    -  `Digital Ocean Kubernetes Service`(DOKS)
    -  `Digital Ocean Managed Database` (In this case Postgres12)
    -  some additional resources (firewall, db-user, db etc.)
  - Helm Chart code for deploying:
    - application `Namespace`
    - application `ConfigMap`
    - application `Secret` (from Helm Secrets)
    - application `Deployment`
    - application `Service` (ClusterIP)
    - NginX ingress controller + NginX `Ingress`
    - CertManager + `Issuer`
    - `HorizontalPodAutoscaler`
  - `Helmfile` for deploying the environment
  - Some instructions and project description

### Terraform

Using Terraform is the best way of managing Cloud resources such as Kubernetes clusters. For this particular task I used Terraform for defining resources in DigitalOcean Cloud with the help of [digitalocean](https://registry.terraform.io/providers/digitalocean/digitalocean/latest/docs) Terraform provider. Instead of providing `secret access key` and `access key id` DigitalOcean gives an option to use `access token` which can be provided via env variable:

```
export DIGITALOCEAN_ACCESS_TOKEN=<token-value>
```

My Terraform configuration also uses `autoupgrade=true` option for the Kubernetes custer. All avaliable versions for the Kubernetes cluster can be described using the `doctl` CLI tool, for example,  woth Docker:

```
docker run -ti --rm -e DIGITALOCEAN_ACCESS_TOKEN  digitalocean/doctl:latest kubernetes options versions
```

> [!NOTE]
> The command works fine only if DIGITALOCEAN_ACCESS_TOKEN is exported as a local env var

After the succesful `terraform apply`, Kubeconfig is copied into `.kube/config`, to use `kubectl`, You need to run:

```
export KUBECONFIG=<path-to-local-config>
```

### Helm

Final `helmcharts` directory structure is:

```
.
├── django-demo
│   ├── Chart.lock
│   ├── charts
│   │   ├── cert-manager-v1.13.1.tgz
│   │   ├── ingress-nginx-4.8.1.tgz
│   │   └── metrics-server-3.11.0.tgz
│   ├── Chart.yaml
│   ├── .helmignore
│   ├── templates
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── _helpers.tpl
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── issuer.yaml
│   │   ├── NOTES.txt
│   │   ├── ns.yaml
│   │   ├── secret.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── service.yaml
│   │   └── tests
│   │       └── test-connection.yaml
│   └── values.yaml
├── helmfile.yaml
└── helm_secrets
    ├── django_config.yaml
    ├── django_secret.yaml
    └── .sops.yaml
```

In addition to standart values, `Chart.yaml` itself  defines dependencies for the Chart to be included:

```
...
dependencies:
- name: ingress-nginx
  version: "4.8.1"
  repository: "https://kubernetes.github.io/ingress-nginx"

- name: cert-manager
  version: "v1.13.1"
  repository: "https://charts.jetstack.io"
  
- name: metrics-server
  version: "3.11.0"
  repository: "https://kubernetes-sigs.github.io/metrics-server/"
```

`ingress-nginx` is the chart for deploying Nginx ingress controller + additional resources to be able to use Ingress

`cert-manager` is the chart for implementing SSL certificates and securing connections (in this case LetsEncrypt is used to issue certificates)

`metrics-server` is the chart for receiving metrics from Kubernetes resiurces (HPA implementation uses metrics for deployment's pods scaling control)

It's important to mention that `cert-manager` chart requires `crds`. [Official Chart Docs](https://artifacthub.io/packages/helm/cert-manager/cert-manager#installing-the-chart) say:

>_"Before installing the chart, you must first install the cert-manager CustomResourceDefinition resources. This is performed in a separate step to allow you to easily uninstall and reinstall
> cert-manager without deleting your installed custom resources."_ 

So it's not the best way to use `installCRDS=true` Chart value. `crds` can be installed via kubectl command:

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.crds.yaml
```

Install Helm Chart requires `ConfigMap` values and `Secret` values. You can define those values via editing personal `values.yaml` file or by using separate `.yaml` files. Values required:

```
databaseURL: <postgres db connection url>
djangoAllowedHosts: <example: "*">
djangoDebug: <"True"/"False">
```

### Helm Secrets

In my case Helm Secrets wrapper was used to decrypt encrypted values in flight.  I needed to generate `GPG` key-pair and create `.sops.yaml` file which points to the fingerpring of the GPG is used.

> [!NOTE]
> Using Helm Secrets requires: Helm Secrets wrapper itself, gpg, sops

To generate key-pair:

```
gpg --gen-key
```
To check avaliable keys:

```
gpg -k
```

Example of `.sops.yaml` file:

```
creation_rules:
  - pgp: <pub-key-fingerprint>
```

Now You can create a file with some secret data (in my case django_secret.yaml with my `databaseURL: <postgres db connection url>`) For encrypting with Helm Secrets:

```
helm secrets encrypt -i <path/to/file>
```

To decrypt:

```
helm secrets decrypt -i <path/to/file>
```

Now You have no need to decrypt a secretfile, just use `helm secrets` instead of `helm` and secret will be decrypted during the installation/upgrading process, for example:

```
cd helmcharts
helm secrets install syndicate django-demo -f helm_secrets/django_secrets.yaml -f helm_secrets/django_config.yaml
```

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/7468565a-6d6a-4e86-add9-589b812b556a" width="450">


```
helm ls
```

![image](https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/d2c512a3-ff57-4981-8dd5-cd42e4eaacda)

### Helmfile 

For this task definition, Helmfile doesn't suit well (in case I defined all sidecharts via dependencies) but it's a powerful tool to configure several Helm installations.

To install Chart via Helmfile:

```
cd helmcharts
helmfile init
helmfile apply
```

### Results

![image](https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/289888ee-543f-474c-977b-22e7a92d6239)

### Implementing PV and PVC to store staticfiles

As [Kubernetes Docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#provisioning) say, there are two ways of `PV` provisioning: `Static` and `Dynamic`.

`Static`: create a `PV` from an existing volume/storage and claim it with `PVC`

`Dynamic`: define `PVC` and `PV` will be created automaticly

Standart Kubernetes actions for `Dynamic` approach:
- The `PVC` requests storage based on the defined requirements, such as size and access mode.
- The `PVC` specifies a `StorageClass` in its configuration. A `StorageClass` defines the provisioning mechanism for dynamic volume provisioning.
- Kubernetes, upon receiving the `PVC` request, checks if a matching `PV` exists. If it doesn't find one, it proceeds to create a new `PV` based on the rules defined in the `StorageClass`.
- Kubernetes uses the DigitalOcean API (or the appropriate cloud provider's API) to provision the actual storage resource based on the `StorageClass` configuration.
- Once the `PV` is created and bound to the `PVC`, the `PVC` is marked as "Bound," and the pod can use the `PVC` as a volume.

DigitalOcean provides `StorageClass` which is the `do-block-storage`. The only problem is that the plugin is still just using plain old block storage volumes under the hood, and those do not support mounting to more than one droplet. So unless that changes, then there's nothing this plugin can do to support `ReadWriteMany`. In this case `ReadWriteOnce` is avaliable but it leads to the _"several Pod replicas are trying to use the same storage block"_ issue. I used `HorizontalPodAutoscaler.spec.minReplicas = 1` just to show the `PVC` implementation.

To implement `PVC` I changed the `Deployment` design, so now I have `InitContainers` in my it.  `entrypoint.sh` changes:

```
#!/bin/sh
set -e
# python manage.py migrate         --> moved to the migrations InitContainer
# python manage.py collectstatic   --> moved to the collectstatic InitContainer

exec gunicorn mysite.wsgi:application \
    --bind 0.0.0.0:8080 \
    --workers 3
```
You can look at the Helm templates:
- for PVC - [pvc.yaml](https://github.com/digitalake/do-terraform-k8s-helm/blob/main/helmcharts/django-demo/templates/pvc.yaml)
- for Deployment - [deployment.yaml](https://github.com/digitalake/do-terraform-k8s-helm/blob/main/helmcharts/django-demo/templates/deployment.yaml)

Lets look at the Helm templating results (PV related code snippets). `pvc.yaml` snippet:

```
# Source: django-demo/templates/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
...
spec:
  storageClassName: do-block-storage
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

I'm using `volumeMode: Filesystem` to be able to mount to the directory.

Corresponding `deployment.yaml` snippet:

```
# Source: django-demo/templates/deployment.yaml
...
initContainers:
        - name: migrations
          image: "ivanopulo/django-demo:1.1-stable"
          envFrom:
          - secretRef:
              name: syndicate-django-demo
          - configMapRef:
              name: syndicate-django-demo
          command: [ python, manage.py, migrate ]
        - name: collectstatic
          image: "ivanopulo/django-demo:1.1-stable"
          volumeMounts:
            - name: staticfiles
              mountPath: /app/staticfiles/
          command: [ python, manage.py, collectstatic, --noinput ]
      containers:
        - name: django-demo
          image: "ivanopulo/django-demo:1.1-stable"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          envFrom:
          - secretRef:
              name: syndicate-django-demo
          - configMapRef:
              name: syndicate-django-demo
          volumeMounts:
            - name: staticfiles
              mountPath: "/app/staticfiles/"
              readOnly: true
              ...
      volumes:
        - name: staticfiles
          persistentVolumeClaim:
            claimName: syndicate-django-demo
```

As we can see, `migrations` conatiner performs migrations, `collectstatic` container is using volume mount to the `/app/staticfiles` directory to perform copying static files. The same volume is being mounted to the `django-demo` container so it will read the staticfiles from it.

> [!NOTE]
> `collectstatic` container has got RW permissions for the volume while `django-demo` container has got only RO configured.

Running `helm upgrade`:

![image](https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/cb1e1c38-1601-42fe-b1c2-897b6e0c3927)

After running helm upgrade we can inspect configured resources.

`PVC`:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/48fd16c3-a837-4e1c-a7ed-bdbcef19630c" width="1000">

`PV`:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/ebd8135d-ec71-4fd0-afee-07eed716f392" width="1000">


`PV` via DigitalOcean Cloud UI:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/392d2279-512f-43b8-b862-bf636573e7ae" width="650">

Mounts for `collectstatic` InitContainer:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/47eb8165-9f4e-4b5e-9d93-981369ce15e9" width="650">

Mounts for `django-demo` application container:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/15e14302-0e22-41c8-a2d2-f351383e89ef" width="650">

`Pods`:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/5a149f1d-e7c3-48b5-9100-ec73bd0bd376" width="650">

`django-demo` application container logs:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/7e0d260e-b741-48f3-9136-3a7ef8f39ed2" width="800">


### Implementing Pod Security

Pod Security Admission labels for namespaces
Once the feature is enabled or the webhook is installed, you can configure namespaces to define the admission control mode you want to use for pod security in each namespace. Kubernetes defines a set of labels that you can set to define which of the predefined Pod Security Standard levels you want to use for a namespace. The label you select defines what action the control plane takes if a potential violation is detected:

- `enforce`	Policy violations will cause the pod to be rejected.
- `audit`	Policy violations will trigger the addition of an audit annotation to the event recorded in the audit log, but are otherwise allowed.
- `warn`	Policy violations will trigger a user-facing warning, but are otherwise allowed.

The Pod Security Standards define three different policies to broadly cover the security spectrum. These policies are cumulative and range from highly-permissive to highly-restrictive. This guide outlines the requirements of each policy.

Profile	Descriptions:

- `Privileged`	Unrestricted policy, providing the widest possible level of permissions. This policy allows for known privilege escalations.
- `Baseline`	Minimally restrictive policy which prevents known privilege escalations. Allows the default (minimally specified) Pod configuration.
- `Restricted`	Heavily restricted policy, following current Pod hardening best practices.

For this task i added `Pod Security Admission` with `enforce` label to enforce pods of the namespace to contain necessary configurations with `Pod Security Standarts Profile` with heavily restricted policy:

Template snippet:
```
apiVersion: v1
kind: Namespace
metadata:
  name: {{ include "django-demo.namespace" . }}
  labels:
    {{- toYaml .Values.namespaceLabels | nindent 4 }}
```

Values snippet:
```
namespaceName: application
...
namespaceLabels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
```

So pods in my `application` namespace will be validated and rejected if don't contain necessary configuration.

Required options for `Restricted` policy are all the options form `Baseline` policy plus these defined in [Kubernetes docs](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted):

- Allowed `Volume` types:

  - spec.volumes[*].configMap
  - spec.volumes[*].csi
  - spec.volumes[*].downwardAPI
  - spec.volumes[*].emptyDir
  - spec.volumes[*].ephemeral
  - spec.volumes[*].persistentVolumeClaim
  - spec.volumes[*].projected
  - spec.volumes[*].secret

- `previlageEscalation` set to "false"
- `runAsNonRoot` set to "true"
- `Running as Non-root user` specified
- `Seccomp` allowed values:
  - RuntimeDefault
  - Localhost
- `Capabilities` Any list of capabilities that includes ALL

For this reason I made some changes to my `Dockerfile` to run as non-root user:
```
...
# Added user creation step
RUN addgroup -S appgroup && adduser -S -u 1001 -G appgroup appuser
...
# Changed owner for the application dir
COPY --chown=appuser:appgroup . /app
...
# Running as non-root
USER appuser
...
```

And pushed new image `django-demo:1.1-non-root` to my DockerHub.

Also I changed the configuration for my `Deployment` to cantain necessary options for all Containers (initContainers too).

Deployment Helm template snippet:

```
...
spec:
...
  template:
  ...
    spec:
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      initContainers:
        - name: migrations
          ...
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          ...
        - name: collectstatic
          ...
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          ...
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
            ...
```

Values snippet:
```
...
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 101
  fsGroup: 101
  seccompProfile:
    type: "RuntimeDefault"

securityContext:
  privileged: false
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
...
```

So now it's unable to deploy `Pod` resource in specified namespace without required configuration by `Restricted` policy.

Checking the user inside the application pod:

![image](https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/710657df-255a-4e8b-9c2f-8a1fdb241206)



### RBAC implementation

For this case i cretaed `ServiceAccount`, `Role` and `RoleBinding` resources as an example:

`ServiceAccount` template snippet:

```
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "django-demo.serviceAccountName" . }}
  namespace: {{ include "django-demo.namespace" . }}
  labels:
    {{- include "django-demo.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  automountServiceAccountToken: {{ .Values.serviceAccount.automount }}
{{- end }}
```

`Role` template snippet:

```
{{- if .Values.serviceAccount.create -}}
{{- if .Values.rbac.create -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "django-demo.serviceAccountName" . }}
  namespace: {{ include "django-demo.namespace" . }}
  labels:
    {{- include "django-demo.labels" . | nindent 4 }}
  {{- with .Values.rbac.rules }}
rules:
  {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
{{- end }}
```

`RoleBinding` template snippet:

```
{{- if .Values.serviceAccount.create -}}
{{- if .Values.rbac.create -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "django-demo.serviceAccountName" . }}
  namespace: {{ include "django-demo.namespace" . }}
  labels:
    {{- include "django-demo.labels" . | nindent 4 }}
subjects:
- kind: ServiceAccount
  name: {{ include "django-demo.serviceAccountName" . }}
  namespace: {{ include "django-demo.namespace" . }}
roleRef:
  kind: Role
  name: {{ include "django-demo.serviceAccountName" . }}
  apiGroup: rbac.authorization.k8s.io
{{- end }}
{{- end }}
```

Values file snippet:
```
serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Automatically mount a ServiceAccount's API credentials?
  automount: false
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

rbac:
  # Specifies whether a rbac role should be created
  create: true
  rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "watch", "list"]
```

My Application doesn't require any special roles so I just specified some as an example.

Pod uses specified Service account:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/7ee33095-4c2f-42fa-b498-209c5f40113b" width="700">

Role details:

<img src="https://github.com/digitalake/do-terraform-k8s-helm/assets/109740456/16ac2476-9eca-45ff-8b4e-691b8a6c9c2e" width="500">












 



