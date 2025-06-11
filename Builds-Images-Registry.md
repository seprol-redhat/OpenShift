# Builds & Images & Registry

- [Builds \& Images \& Registry](#builds--images--registry)
  - [Referencias](#referencias)
  - [Registro de imagens integrado ao OpenShift](#registro-de-imagens-integrado-ao-openshift)
  - [Configurar acesso externo ao registro](#configurar-acesso-externo-ao-registro)
  - [Configuring image registry settings](#configuring-image-registry-settings)
  - [Managing Image Streams and Tags](#managing-image-streams-and-tags)
    - [Triggering updates on image stream changes](#triggering-updates-on-image-stream-changes)
  - [Catalog](#catalog)
  - [Dockerfile exemplo](#dockerfile-exemplo)
  - [Push by Podman](#push-by-podman)
  - [Push by oc command](#push-by-oc-command)
  - [Push by build on OpenShift](#push-by-build-on-openshift)
  - [Pruning images](#pruning-images)
  - [Deploy exemplo](#deploy-exemplo)
    - [Trigger update](#trigger-update)

## Referencias

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/index
- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/images/index

## Registro de imagens integrado ao OpenShift

- Localização:
  - Administrator -> Builds -> ImageStreams
  - Developer -> Search -> ImageStream

- Endereço interno: `image-registry.openshift-image-registry.svc:5000`

- As imagens podem ser armazenas em qualquer namespace
  - É recomendado que as imagens dos Apps estejam no mesmo namespace do App

## Configurar acesso externo ao registro

- Por padrão, o acesso externo não é configurado

- É criada uma rota via Custom Resource: `configs.imageregistry.operator.openshift.io`
- Endereço externo: `default-route-openshift-image-registry.apps.dev.labredhat.seprol`

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/securing-exposing-registry#registry-exposing-default-registry-manually_securing-exposing-registry

## Configuring image registry settings

- Para o caso de builds no OpenShift, podem ser necessárias algumas configurações adicionais no Custom Resource: `image.config.openshift.io/cluster`

- Exemplo:

  - Registros permitidos
  - Registros bloqueados

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/images/image-configuration#images-configuration-file_image-configuration

## Managing Image Streams and Tags

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/images/managing-image-streams

### Triggering updates on image stream changes

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/images/triggering-updates-on-imagestream-changes

## Catalog

- https://catalog.redhat.com/en

## Dockerfile exemplo

```Dockerfile
FROM registry.access.redhat.com/ubi9/ubi

RUN yum -y install vim && yum clean all -y

ENV VERSION=1

CMD ["sleep", "infinity"]
```

## Push by Podman

```sh
podman build -t custom-ubi9:<tag> .

oc project <project>

podman login --tls-verify=false -u=<user> -p=$(oc whoami -t) default-route-openshift-image-registry.apps.dev.labredhat.seprol

podman push --tls-verify=false localhost/custom-ubi9:<tag> default-route-openshift-image-registry.apps.dev.labredhat.seprol/<project>/custom-ubi9:<tag>
# --remove-signatures
```

## Push by oc command

- Não funciona para imagens locais
- A imagem tem que estar em um registro com acesso externo

```sh
oc project <project>

#oc create imagestream <new-imagestream-name>
oc import-image rhel9/nodejs-22-minimal:9.6-1749013782 --from=registry.redhat.io/rhel9/nodejs-22-minimal:9.6-1749013782 --confirm
```

## Push by build on OpenShift

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/builds_using_buildconfig/custom-builds-buildah#builds-build-custom-builder-image_custom-builds-buildah

```sh
oc new-build \
  --binary \
  --strategy=docker \
  --name=ubi9 \
  -l "app=ubi9" \
  --to="image-registry.openshift-image-registry.svc:5000/<project>/custom-ubi9:latest"

oc start-build ubi9 --from-dir=. -w -F

oc tag custom-ubi9:latest custom-ubi9:<tag>
```

- Pruning builds
  - https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/building_applications/pruning-objects#pruning-builds_pruning-objects

## Pruning images

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/building_applications/pruning-objects#pruning-images_pruning-objects

```sh
oc adm prune images --keep-tag-revisions=3 --keep-younger-than=0 --registry-url=https://default-route-openshift-image-registry.apps.dev.labredhat.seprol --confirm -n <project>
```

## Deploy exemplo

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: ubi9-testes-<n>
  labels:
    app: ubi9-testes
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ubi9-testes
  template:
    metadata:
      labels:
        app: ubi9-testes
    spec:
      containers:
        - name: ubi9-testes
          command:
            - sh
            - "-c"
            - echo The app is running! && printenv && sleep infinity
          image: image-registry.openshift-image-registry.svc:5000/<ns>/custom-ubi9:<n>
```

### Trigger update

- Adicionar ao deployment

```yaml
metadata:
  annotations:
    image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"custom-ubi9:latest"},"fieldPath":"spec.template.spec.containers[?(@.name==\"ubi9-testes\")].image"}]'
```
