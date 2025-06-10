# Builds & Images & Registry

- [Builds \& Images \& Registry](#builds--images--registry)
  - [Referencias](#referencias)
  - [Integrated OpenShift image registry](#integrated-openshift-image-registry)
  - [Exposing a default registry manually](#exposing-a-default-registry-manually)
  - [Configuring image registry settings](#configuring-image-registry-settings)
  - [Managing Image Streams and Tags](#managing-image-streams-and-tags)
    - [Triggering updates on image stream changes](#triggering-updates-on-image-stream-changes)
  - [Dockerfile exemplo](#dockerfile-exemplo)
  - [Push by Podman](#push-by-podman)
  - [Push by oc command](#push-by-oc-command)
  - [Push by build on OpenShift](#push-by-build-on-openshift)
  - [Pruning images](#pruning-images)
  - [Opções](#opções)
  - [Comandos gerais](#comandos-gerais)
  - [Deploy exemplo](#deploy-exemplo)

## Referencias

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/index
- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/images/index

## Integrated OpenShift image registry

- O namespace `openshift` armazenas diversas imagens criadas a partir da instalação
- Localize as imagens em: Builds -> ImageStreams
- Endereço interno: `image-registry.openshift-image-registry.svc:5000`

- As imagens podem ser armazenas em qualquer namespace

## Exposing a default registry manually

- External access by exposing it with a route
- Endereço externo: `default-route-openshift-image-registry.apps.dev.labredhat.seprol`
- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/securing-exposing-registry#registry-exposing-default-registry-manually_securing-exposing-registry

## Configuring image registry settings

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/images/image-configuration#images-configuration-file_image-configuration

## Managing Image Streams and Tags

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/images/managing-image-streams

### Triggering updates on image stream changes

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/images/triggering-updates-on-imagestream-changes

## Dockerfile exemplo

```Dockerfile
FROM registry.access.redhat.com/ubi9/ubi

RUN yum -y install vim && yum clean all -y

#ADD myfile /test/myfile

#RUN chgrp -R 0 /some/directory && \
#    chmod -R g=u /some/directory

CMD ["sleep", "infinity"]
```

## Push by Podman

```sh
podman build -t ubi9/ubi:<tag> .

oc project <project>

podman login --tls-verify=false -u=<user> -p=$(oc whoami -t) default-route-openshift-image-registry.apps.dev.labredhat.seprol

podman push --tls-verify=false --remove-signatures localhost/ubi9/ubi:<tag> default-route-openshift-image-registry.apps.dev.labredhat.seprol/<project>/ubi9:<tag>
```

## Push by oc command

- Não funciona para imagens em localhost

```sh
podman pull registry.access.redhat.com/ubi9
podman images

oc project <project>

#oc create imagestream <new-imagestream-name>
oc import-image ubi9:1 --from=registry.access.redhat.com/ubi9 --confirm
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
  --to="image-registry.openshift-image-registry.svc:5000/<project>/ubi9:<tag>"

oc start-build ubi9 --from-dir=. -w -F
```

- Pruning builds
  - https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/building_applications/pruning-objects#pruning-builds_pruning-objects

## Pruning images

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/building_applications/pruning-objects#pruning-images_pruning-objects

```sh
oc adm prune images --keep-tag-revisions=2 --keep-younger-than=0 --registry-url=https://default-route-openshift-image-registry.apps.dev.labredhat.seprol --confirm -n <project>
```

## Opções

```sh
--scheduled=true
```

## Comandos gerais

```sh
oc get istag
```

## Deploy exemplo

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: ubi9-testes-<n>
  namespace: <ns>
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
            - echo The app is running! && sleep infinity
          image: image-registry.openshift-image-registry.svc:5000/<ns>/ubi9:<n>
          ports:
            - containerPort: 80
              protocol: TCP
```
