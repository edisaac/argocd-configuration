# kubernetes-configuration

## Ingresar y Listar Proyectos proyecto Gcloud

```shell
gcloud auth login
gcloud projects list
```



## CREAR CLUSTER QA

#### Seleccionar el proyecto edisaac QA

```shell
gcloud config set project dockers-edisaac-tst
```

#### Crear Cluster de qa-cluster 

```bash
gcloud container clusters create qa-cluster \
--num-nodes=1 \
--machine-type=e2-standard-4 --zone us-east1-b \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform" \
--cluster-version 1.20  \
--enable-ip-alias 
```

#### Conectar al proyecto edisaac QA

```shell
gcloud container clusters get-credentials qa-cluster --zone us-east1-b --project dockers-edisaac-tst
```

#### Crear namaespace qa

```shell
kubectl create ns qa
```

#### Crear configuracion de docker registry

```bash
kubectl create secret docker-registry regcred --docker-server=registry.gitlab.com --docker-username=<<USER>> --docker-password=<<PASS> -n qa
```

##### crear y ver ip statica para ingress para QA

```shell
gcloud compute addresses create qa-static-ip-ingress  --region=us-east1

gcloud compute addresses describe qa-static-ip-ingress  --region=us-east1 \
  --format='value(address)'
```

##### Modificar el loadBalancerIP en la configuracion de QA

con el ip creado en el branch **RELEASE** de **developers.edisaac/devops/kubernetes-configuration.git**

modificar el archivo **/qa/config/ingress/traefik.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  loadBalancerIP: <<qa-static-ip-ingress>>
```

#### configurar los secretos para certificado https 

```shell
kubectl create secret tls edisaac-tls-secret -n qa \
  --cert=<<ruta local del archivo crt>> \
  --key=<<ruta local del archivo key>> 
  
```

ejemplo

```shell
kubectl create secret tls edisaac-tls-secret -n qa  \
  --cert=/persistence/app/traefik/conf/dinamic/certs/wildcard_edisaac_2021_2022.crt \
  --key=/persistence/app/traefik/conf/dinamic/certs/wildcard_edisaac_2021_2022_private.key
```

Crear un disco para el traefik

```shell
gcloud compute disks create --size=10GB --zone=us-east1-b traefik-persistent
```



## Instalar Argos en cluster QA

##### Conectar al proyecto edisaac PROD

```shell
gcloud container clusters get-credentials qa-cluster --zone us-east1-b --project dockers-edisaac-tst
```

### Install Argos

```shell
kubectl config get-contexts
kubectl config use-context <<QA CONTEXT NAME>>
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Ver documentacion en https://argoproj.github.io/argo-cd/getting_started/#1-install-argo-cd

### Install Argos CLI

```shell
brew install argocd
```

Ver documentacion en https://argoproj.github.io/argo-cd/cli_installation/#linux-and-wsl

Ver documentacion de instalar brew https://www.codegrepper.com/code-examples/shell/install+brew+ubuntu+20.04

### Modificar la imagen para que no use TLS interno

Agregar opcion insecure al deploy de argocd-server

```shell
kubectl patch  deployment argocd-server -n argocd  --type json -p='[{"op": "add", "path": "/spec/template/spec/containers/0/command/3", "value": "--insecure"}]'
```

Ejemplo de como quitar la linea insecure

kubectl patch  deployment argocd-server -n argocd  --type json -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/command/3"}]'

### Abrir puerto local a argocd

```shell
kubectl port-forward svc/argocd-server -n argocd 8888:80
```

### Ingresar a argo con usuario admin

```shell
argocd login 127.0.0.1:8888
```

para  obtener la contraseña inicial

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

para cambiar de contraseña ver https://argoproj.github.io/argo-cd/getting_started/#4-login-using-the-cli

### Registrar repositorio usando ssh keys

##### Configuracion de gitlab

Generar los keys seguir el manual de gitlab https://docs.gitlab.com/ee/ssh/#generate-an-ssh-key-pair

Registrarlo en gitlab con el usuario que tenga permisos sobre el respositorio de developers.edisaac/devops/kubernetes-configuration   https://docs.gitlab.com/ee/ssh/#generate-an-ssh-key-pair

##### Configurar repo en kubernetes-configuration con shh key

```shell
argocd repo add git@github.com:edisaac/kubernetes-configuration.git --name  kubernetes-configuration --ssh-private-key-path <<ruta local del archivo key>>
```

##### Configurar repo en argocd-configuration con shh key

```shell
argocd repo add git@github.com:edisaac/argocd-configuration.git --name  argocd-configuration --ssh-private-key-path <<ruta local del archivo key>>
```

 

### Listar los nombres de los cluster

```shell
kubectl config get-contexts
```

### Agregar el cluster NAME de QA

```shell
argocd cluster add <<CLUSTER NAME QA>> --name qa-cluster
```

### Aplicar el Application Argos Inicial

```shell
kubectl apply -f ./qa/inicial.yaml
```





