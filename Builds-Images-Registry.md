# Builds & Images & Registry

- [Builds \& Images \& Registry](#builds--images--registry)
  - [Referencias](#referencias)
  - [Namespace padrão](#namespace-padrão)
  - [Server - Registry](#server---registry)
  - [Exposing a default registry manually](#exposing-a-default-registry-manually)
  - [Dockerfile exemplo](#dockerfile-exemplo)
  - [By Podman](#by-podman)
  - [By oc command](#by-oc-command)
  - [By build on OpenShift](#by-build-on-openshift)
  - [Pruning images](#pruning-images)
  - [Opções](#opções)
  - [Comandos gerais](#comandos-gerais)
  - [Deploy exemplo](#deploy-exemplo)

## Referencias

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/index

## Namespace padrão

- `openshift`

## Server - Registry

- image-registry.openshift-image-registry.svc:5000
- default-route-openshift-image-registry.apps.dev.labredhat.seprol

## Exposing a default registry manually

- https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/registry/securing-exposing-registry#registry-exposing-default-registry-manually_securing-exposing-registry

## Dockerfile exemplo

```Dockerfile
FROM registry.access.redhat.com/ubi9/ubi

RUN yum -y install vim && yum clean all -y

#ADD myfile /test/myfile

#RUN chgrp -R 0 /some/directory && \
#    chmod -R g=u /some/directory

CMD ["sleep", "infinity"]
```

## By Podman

```sh
podman build -t ubi9/ubi:<tag> .

oc project <project>

podman login --tls-verify=false -u=<user> -p=$(oc whoami -t) default-route-openshift-image-registry.apps.dev.labredhat.seprol

podman push --tls-verify=false --remove-signatures localhost/ubi9/ubi:<tag> default-route-openshift-image-registry.apps.dev.labredhat.seprol/<project>/ubi9:<tag>
```

## By oc command

- Não funciona para imagens em localhost

```sh
podman pull registry.access.redhat.com/ubi9
podman images

oc project <project>

#oc create imagestream <new-imagestream-name>
oc import-image ubi9:1 --from=registry.access.redhat.com/ubi9 --confirm
oc import-image rhel9/nodejs-22-minimal:9.6-1749013782 --from=registry.redhat.io/rhel9/nodejs-22-minimal:9.6-1749013782 --confirm
```

## By build on OpenShift

- https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/builds_using_buildconfig/custom-builds-buildah#builds-build-custom-builder-image_custom-builds-buildah

```sh
oc new-build \
  --binary \
  --strategy=docker \
  --name=ubi9 \
  -l "app=ubi9" \
  --to="image-registry.openshift-image-registry.svc:5000/<project>/ubi9:<tag>"

oc start-build ubi9 --from-dir=. -w -F
```

## Pruning images

- https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/building_applications/pruning-objects#pruning-images_pruning-objects

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
