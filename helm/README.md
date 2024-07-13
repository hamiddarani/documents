# Helm

```bash
helm completion bash > /etc/
rm ~/.cache/helm
rm -rf .config/helm
```

**commands**

```bash
helm search hub <package-name> # search in artifacthub.io
helm search repo <package-name> # search in repository
helm repo add <repository-name> <repository-url> # add repository
helm repo remove <repository-name> # remove repository
helm repo list # list all repositories
cat ~/.config/helm/repositories.yaml # list of repositories
cat ~/.config/helm/repository/bitnami-index.yaml # all packages with information
helm repo update # update all indices cache
helm install -n <namespace> <release-name> <repo-name> --version <version> --creaet-namespace
helm uninstall <release-name> -n <namespace-name> --keep-history
helm status <release-name> -n <namespace-name>
helm delete <release-name> -n <namespace-name>
helm list -A # --all-namespaces
helm ls -A -a
helm ls -n <namespace-name>
helm upgrade <release-name> <repository-name> -n <namespase-name> --values <values.yaml path>
helm history <release-name> -n <namespace-name>
helm rollback <release-name> <release-number> -n <namespace-name> 
helm get values -n <namespace-name> <release-name>
helm get manifest -n <namespace-name> <release-name>
```

```bash
helm show readme bitnami/redis
helm show crds bitnami/redis # kubectl get apiservices
helm show chart bitnami/redis
helm show values bitnami/redis
```

**helm plugins**

```bash
helm plugin install <repository-url>
helm diff revsion -n <namespace-name> <revision-number1> <revision-number2>
helm diff release -n <namespace-name> <release-name-1> <release-name-2>
```

**set values**

```bash
helm install metallb bitnami/metallb -n metallb-system --create-namespace --set "name=metallb"

--set "key=value" # string|number
--set "key=value" --set "key=value" OR --set "key=value,key=value"
--set "master.count=value" # object
--set "items={a,b,c}" # [a,b,c]
--set "tolerations[0].operator=Exists"
--set "nodeSelector.kubernetes\.io/role=master"
helm template metallb bitnami/metallb -n metallb-system --set "controller.tolerations[0].operator=Exists"

helm template metallb bitnami/metallb -n metallb-system -f metallb-values.yaml
```

**helm with kustomize**

```
helm repo add <argo-repo>
helm istall argocd argo/argo-cd -n argocd
```

```tex
helm template argocd argo/argo-cd -n argocd > argo-manifests.yaml
touch kustomization.yaml
nano kustiomization.yaml

apiVersion: kustomize.config.k8s/v1beta1
kind: Kustomization
resourses:
  - argo-manifests.yaml
patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: argocd-repo-server
    patch: |-
      - path: /spec/template/spec/hostIPC
        value: true
        op: add

kustomize build
```

**add repository config**

```yaml
# repositories.yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-github-repo-anakonda
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/hamiddarani/anakonda.git
  
---
apiVersion: kustomize.config.k8s/v1beta1
kind: Kustomization
namespace: argocd
resourses:
  - argo-manifests.yaml
  - repositories.yaml
patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: argocd-repo-server
    patch: |-
      - path: /spec/template/spec/hostIPC
        value: true
        op: add
```

```bash
kubectl apply -k .
```

**create custom package**

```tex
mkdir testapp
touch Chart.yaml

apiVersion: v2 # helm3
name: testapp
version: 0.0.1
appVersion: 1.0.0
type: application
description: a simple helm chart to deploy testapp

helm lint .

helm create <helm-chart-name>
```

```yaml
{{ if .Capabilities.APIVersions.Has "external-secrets.io/v1beta1" }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}-{{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
  labels:
    version: {{ .Values.version | quote }}
    last-change-at: {{ now | quate }}
    owner: {{ .Values.owner | default "none" | quote }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app.kubernetes.io/name: <pod-labels>
  template:
    metadata:
      labels:
        app.kubernetes.io/name: <pod-labels>
    spec:
      containers:
        - name: <container-name>
          image: <container-name>
          ports:
            - containerPort: 80
          envFrom:
            configMapRef:
              name: <configmap-name>
{{ end }}        
```

```
# values.yaml
replicas: 1

```

**conditions**

```tex
{{- if gt (int Capabilities.KubeVersion.Minor) 29 -}}

{{- else -}}

{{- end -}}

{{- if or (gt (int Values.replicas) 1) (and (eq (int .Values.replicas) 1) (eq .Values.DeploymentType "statefulset")) -}}
...
{{- end -}}
```

**NOTES.txt**

```tex
Hello, welcome to testapp.

Your application deployed with the release name = {{ .Release.Name }} in {{ .Release.Namespace }}

Enjoy using testapp.
```

**helpers.tpl**

```yaml
{{- define applicationLabels }}
app.kubernetes.io/name: {{ Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/created-by: hamid
{{- end -}}
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <configmap-name>
  labels:
    {{- include "applicationLabels" | nindent 4 }}
data:
  FETCH_CLUSTER_DATA: |
    {{ (lookup "v1" "Secret" "kube-system" "vpa-tls-certs").data }}
```

**lookup**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <configmap-name>
data:
  FETCH_CLUSTER_DATA: |
    {{ (lookup "v1" "Secret" "kube-system" "vpa-tls-certs").data }}
```

**Files**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test
data:
  my-config: |-
    {{ .Files.Get "config.yaml" }}
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test
data:
{{ (.Files.Glob "*.conf").AsConfig | indent 2 }}
```

**with**

```yaml
spec:
  {{- with .Values.tolerations }}
  tolerations:
  {{- toYaml . | nindent 8 }}
  {{- end }}
```

**range**

```yaml
repositories:
  - url: ...
    name: ...
```

```
{{- .Values.repositories -}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "name" . }}-{{ .name }}
  labels:
    {{- include "applicationLabels" . | nindent 4 }}
stringData:
  url: {{ .url }}
{{- end -}}
```

**package**

```bash
helm package testapp
helm repo index
```

