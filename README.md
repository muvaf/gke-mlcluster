# MLCluster API Type

This repo exists to serve as a feedback for [composite resource work](https://github.com/crossplane/crossplane/pull/1163).

You can install this via
```bash
kubectl crossplane stack install --cluster -n crossplane-system 'muvaf/gke-mlcluster:v0.0.1' mlcluster
```

Try it via:
```bash
kubectl create -f example.yaml
```

Note that template stacks aimed at bundling applications rather than serve as resource classes for composite resources.
I just wanted to see how it'd look like if one tries to achieve a similar goal with template stacks so that we can see the shortcomings, developer experience and user experience.

### No Dynamic Provisioning

* Cannot use existing Crossplane claims.
  * Claims do not have `providerRef` that the managed resources here can fetch and use for provisioning.
  * Normally this is supplied by resource classes but in our case, they are embedded in the package. So, you have to put `providerRef` on `MLCluster` resource directly.

* No binding mechanism.
  * There is no scheduling controller to bind a claim to `MLCluster`. Though implementation of this wouldn't be very cumbersome.

By binding I mean a bond between `KubernetesCluster` and `MLCluster`. Because the `GKECluster` that is provisioned by templating-controller can be picked up by a `KubernetesCluster` claim, mimicking the classic static provisioning case. However, it'd not be as useful since `MLCluster` resource actively reconciles its children, meaning even if the claim wanted to delete it, it'd get recreated.

* Resource classes are embedded in the package.
  * You can see that we prescribe how the resources look like in [kustomize](kustomize) folder.
  * These are packaged and are not visible to the user until they create an `MLCluster` resource or go and look at the git repo.

* No secret propagation to the namespace of `MLCluster` (if it was namespaced).
  * This is probably doable but currently not possible.

### Authoring Experience

* It's really verbose to achieve simple things.
  * This is fine for apps since people are already writing helm charts and kustomizations but I think they are overkill for simple bindings.
  * See [kustomization.yaml](kustomize/kustomization.yaml) and [kustomizeconfig.yaml](kustomize/kustomizeconfig.yaml) to see things you have to do to handle `providerRef`, which could easily be handled by an opinionated Crossplane tool.
  
* Raw CRD is supplied by the user.
  * I have used a scheme-less CRD but it's also possible for user to write their own validations. I'm not sure how much improvement can be achieved here.
  * Maybe generate a validation looking at bindings in [behavior.yaml](.registry/behavior.yaml), browse the CRDs and copy validation rules of those specific fields?
    * Since all fields are known, a tool can do this easily during the beginning of the runtime. This may not be as doable with existing composite resource work since there are N composite resource classes using the same CRD for possibly different fields and new ones can be added at any time.


These are my first impressions. Keep in mind that template stacks were not meant to serve that purpose anyway but I think most of the things listed could be somehow made possible though there are conceptual differences.

The biggest one is the packaging. Template stacks are, well, **stacks**. A lot of things are packed in an image that you cannot touch unless you deploy a new one, which could be a desirable property but it's just not as flexible as resource classes we have today.

The second one is visibility of what you're going to get if you choose to provision an `MLCluster`. In case of apps, you might not care about N Roles, M Deployments and PVCs and all those resources. However, what composite resources aim to achieve is to enable users to provision additional helper resources that are configured together with one main resource. So, one can expect that you won't get an array of resources longer than like 10. So, while so much visibility for apps could be too verbose, it's preferred when you have relatively small number of resources that constructs one entity(`GKECluster`+`NodePools` in this case). 

I think what we have as template stacks serve its purpose for apps and ready-made environments well and improvements are being made. It's somewhat possible to stretch it to cover more cases but then we risk ending up with something that is too generic. On the other hand, it might be interesting to see how apps can be represented in a PVC/PV model and how the experience would look like. So, I can see how we might want to investigate a convergence possibility but it's way too early for that since both composite resources and template stacks are very new and we all know what is the [root of all evil](https://wiki.c2.com/?PrematureOptimization).
