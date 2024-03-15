# microservice-api-spring-boot-config
K8s config for https://github.com/jonashackt/microservice-api-spring-boot


### Configuration with Kustomize

We're using [Kustomize](https://kustomize.io/) to make configuration through CI systems easier (see https://stackoverflow.com/questions/71704023/how-to-use-kustomize-to-configure-traefik-2-x-ingressroute-metadata-name-spec).

Configuring the Deployment, Service and IngressRoute to use a branch like `great-feature-branch` works like this:

```
RULE_HOST_BRANCHNAME=great-feature-branch

cat > ./ingressroute-patch.yml <<EOF
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: microservice-api-spring-boot-ingressroute
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - match: Host(\`microservice-api-spring-boot-$RULE_HOST_BRANCHNAME.tekton-argocd.de\`)
      kind: Rule
      services:
        - name: microservice-api-spring-boot
          port: 80

EOF

kustomize edit set namesuffix -- -great-feature-branch
kustomize edit set label branch:great-feature-branch
kustomize edit set image registry.gitlab.com/jonashackt/microservice-api-spring-boot:great-feature-branch
kustomize build .
```




### How it all works

We have [a EKS cluster running with Traefik deployed in CRD style (full setup on GitHub](https://github.com/jonashackt/tekton-argocd-eks)) and wan't to deploy our app https://gitlab.com/jonashackt/microservice-api-spring-boot with the Kubernetes objects Deployment, Service and IngressRoute (see [configuration repository here](https://gitlab.com/jonashackt/microservice-api-spring-boot-config)). The manifests look like this:

`deployment.yml`:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: microservice-api-spring-boot
    spec:
      replicas: 3
      revisionHistoryLimit: 3
      selector:
        matchLabels:
          app: microservice-api-spring-boot
          branch: main
      template:
        metadata:
          labels:
            app: microservice-api-spring-boot
            branch: main
        spec:
          containers:
            - image: registry.gitlab.com/jonashackt/microservice-api-spring-boot:c25a74c8f919a72e3f00928917dc4ab2944ab061
              name: microservice-api-spring-boot
              ports:
                - containerPort: 8098
          imagePullSecrets:
            - name: gitlab-container-registry

`service.yml`:

    apiVersion: v1
    kind: Service
    metadata:
      name: microservice-api-spring-boot
    spec:
      ports:
        - port: 80
          targetPort: 8098
      selector:
        app: microservice-api-spring-boot
        branch: main

`traefik-ingress-route.yml`:

    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: microservice-api-spring-boot-ingressroute
      namespace: default
    spec:
      entryPoints:
        - web
      routes:
        - match: Host(`microservice-api-spring-boot-BRANCHNAME.tekton-argocd.de`)
          kind: Rule
          services:
            - name: microservice-api-spring-boot
              port: 80

We already use [Kustomize](https://kustomize.io/) and especially the `kustomize` CLI (on a Mac or in GitHub Actions install with `brew install kustomize`) with the following folder structure:

    ├── deployment.yml
    ├── kustomization.yaml
    ├── service.yml
    └── traefik-ingress-route.yml

Our `kustomization.yaml` looks like this:

    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    
    resources:
    - deployment.yml
    - service.yml
    - traefik-ingress-route.yml
    
    images:
    - name: registry.gitlab.com/jonashackt/microservice-api-spring-boot
      newTag: foobar
    
    commonLabels:
      branch: foobar
    
    nameSuffix: foobar

Now changing the `metadata.name` dynamically to add a suffix to the Deployment's, Service's and IngressRoute's `.metadata.name` from within our GitHub Actions workflow is easy with `kustomize` CLI (because we want the suffix to use a prefixed `-`, we need to use the `-- -barfoo` syntax here):

    kustomize edit set namesuffix -- -barfoo

Check the result with

    kustomize build .

Also changing the `.spec.selector.matchLabels.branch`, `.spec.template.metadata.labels.branch` and `.spec.selector.branch` in the Deployment and Service is no problem:

    kustomize edit set label branch:barfoo

Changing the `.spec.template.spec.containers[0].image` of our Deployment works with:

    kustomize edit set image registry.gitlab.com/jonashackt/microservice-api-spring-boot:barfoo

**But looking into our `IngressRoute` it seems that `.spec.routes[0].services[0].name` and `.spec.routes[0].match = Host()` can't be changed with Kustomize out of the box?! So how can we change both fields without the need for a replacement tooling like yq or even `sed`/ `envsubst`?**


**1. Change the `IngressRoute`s `.spec.routes[0].services[0].name` with Kustomize**
------------------------------------------------------------------------

Changing the `IngressRoute`s `.spec.routes[0].services[0].name` is possible with Kustomize using [a `NameReference` transformer (see docs here)](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/transformerconfigs/README.md#name-reference-transformer). Therefore we need to include the `configurations` keyword in our `kustomize.yaml`:

    nameSuffix: foobar
    configurations:
      # Tie target Service metadata.name to IngressRoute's spec.routes.services.name
      # Once Service name is changed, the IngressRoute referrerd service name will be changed as well.
      - nameReference.yml

We also need to add file called `nameReference.yml`:

    nameReference:
      - kind: Service
        fieldSpecs:
          - kind: IngressRoute
            path: spec/routes/services/name

As you can see we tie the Service's `name` to the IngressRoutes `spec/routes/services/name`. Now running

    kustomize edit set namesuffix barfoo

will not only change the `metadata.name` tags of the Deployment, Service and IngressRoute - but also the `.spec.routes[0].services[0].name` of the IngressRoute, since it is now linked to the `metadata.name` of the Service. Note that this only, if both the referrer and the target's have a `name` tag.


**2. Change a part of the IngressRoutes `.spec.routes[0].match = Host()**
---------------------------------------------------------------------

The second part of the question ask how to change a part of the IngressRoutes `.spec.routes[0].match = Host()`. There's [an open issue in the Kustomize GitHub project](https://github.com/kubernetes-sigs/kustomize/issues/4012). Right now Kustomize doesn't support this use case - only writing a custom generator plugin for Kustomize. As this might not be a preferred option, there's another way inspired by [this blog post](https://traefik.io/blog/deploy-traefik-proxy-using-flux-and-gitops/). As we can create yaml files inline in our console using the syntax `cat > ./myyamlfile.yml <<EOF ... EOF` we could also use the inline variable substitution.

So first define the branch name as variable:

    RULE_HOST_BRANCHNAME=foobar

And then use the described syntax to create a `ingressroute-patch.yml` file inline:

    cat > ./ingressroute-patch.yml <<EOF
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: microservice-api-spring-boot-ingressroute
      namespace: default
    spec:
      entryPoints:
        - web
      routes:
        - match: Host(\`microservice-api-spring-boot-$RULE_HOST_BRANCHNAME.tekton-argocd.de\`)
          kind: Rule
          services:
            - name: microservice-api-spring-boot
              port: 80
    
    EOF

The last step is to use the `ingressroute-patch.yml` file as `patchesStrategicMerge` inside our `kustomization.yaml` like this:

    patchesStrategicMerge:
      - ingressroute-patch.yml

Now running `kustomize build .` should output the correct Deployment, Service and IngressRoute for our setup:

    apiVersion: v1
    kind: Service
    metadata:
      labels:
        branch: barfoo
      name: microservice-api-spring-boot-barfoo
    spec:
      ports:
      - port: 80
        targetPort: 8098
      selector:
        app: microservice-api-spring-boot
        branch: barfoo
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        branch: barfoo
      name: microservice-api-spring-boot-barfoo
    spec:
      replicas: 3
      revisionHistoryLimit: 3
      selector:
        matchLabels:
          app: microservice-api-spring-boot
          branch: barfoo
      template:
        metadata:
          labels:
            app: microservice-api-spring-boot
            branch: barfoo
        spec:
          containers:
          - image: registry.gitlab.com/jonashackt/microservice-api-spring-boot:barfoo
            name: microservice-api-spring-boot
            ports:
            - containerPort: 8098
          imagePullSecrets:
          - name: gitlab-container-registry
    ---
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      labels:
        branch: barfoo
      name: microservice-api-spring-boot-ingressroute-barfoo
      namespace: default
    spec:
      entryPoints:
      - web
      routes:
      - kind: Rule
        match: Host(`microservice-api-spring-boot-barfoo.tekton-argocd.de`)
        services:
        - name: microservice-api-spring-boot-barfoo
          port: 80
