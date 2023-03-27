# Tricks for enforcing conventions for your Kubernetes cluster using only YAML

Have you ever operated a Kubernetes cluster for multiple developers? If you have, you probably realized quickly that things are going to be a lot smoother if you could just enforce some basic conventions. Maybe all your services have a well defined endpoint for the liveness probe but developers sometimes forget to set it up. Or maybe developers should always use a semantic version tag on their containers and avoid :latest. Or maybe there is a deprecated Kubernetes API field and you'd like to ensure it is never used in your cluster. In this talk we will run through a series of easy solutions to help enforce conventions using only YAML. You have a lot more control that you might realize. Learn from a Kubernetes contributor involved in the development of numerous extensibility features including CRDs, admission webhooks and admission policies. We will show you some handy tricks and leveraging new features like Validating Admission Policies alpha API introduced in 1.26.

# Outline

- Introduce topic
  - We are going to enforce non-trivial cluster conventions entirely in YAML.
    No new controllers or webhooks. No 3rd party system installations.
  - We will be using "stock k8s" but with 1.27 alpha features
  - Key primitives we will combine:
    - ValidatingAdmissionPolicies (show example)
    - RBAC, in particular the use of custom verbs and resources
    - CRDs, but maybe not in the way your are used to
      - We will also make use of CRD Validation rules (show example)

- Motivate namespace examples (much ado about namespace labels)
  - We're going to use some simplified examples to show a series of capabilities
  - We're going to imagine that we've got a multi-purpose cluster for a small
    org with mutiple development teams
  - We're going to imagine that we want to make better use of namepsaces and
    labels to help organize things
- Punch through examples

- Motivate pod policy examples
  - We're not limited to labels, we can operate on any objet fields
  - The examples here are classic k8s policies that all major policy engines
    support. So you might not ever need to write policies like these directly
    if you adopt a policy engine, but it's good to see how they work.
- Show examples

- Show rollout and ratcheting as modifications to prior examples
  - Warning and audit use cases (TODO)
    - Note that audit data can be processed using a audit backend to
      publish policy violations in any way desired
  - Enabling new policies on a cluster can be tricky, since existing
    API clients might not yet be respecting the policy. Warnings and
    audit annotations are good tools to use before setting a policy
    to enforcing. Bindings are helpful because they make it possible
    to incrementally rollout a change.
      TODO: Example of using multiple bindings.

- Key closing comments
  - Even if you never plan to write your own controller, you still
    can define your own CRDs to help manage your cluster
  - RBAC pairs very well with CEL admission control
    - synthetic resources and custom verbs can be interpreted 
    - authz of fine grained operations such as field level changes
      are possible
    - Visualization: CEL connecting CRDs and RBAC to create a new class of extensibility
  - Policy engines become even more important
    - Reduced dependance on webhooks makes policy easier and safer, this should
      enable a larger ecosystem.  Many policy bundles should be possible 
      Visualization: Before/after of how policy engines can integrate with k8s
    - Policy engines are essential for safe rollout, defining and managing policy
      bundles, and enforcing compilance across multiple enforcement points

## Commands

```sh
## Namespaces

# First let's imagine we want an environment label on our namespaces, and we want to control what values it can be set to.
$ kubectl apply -f validate-environment-label.yaml
$ kubectl apply -f ns-invalid-environment.yaml
The namespaces "example-invalid-environment" is invalid: : ValidatingAdmissionPolicy 'production-namespace-policy.k8s.jpbetz.github.io' with binding 'production-namespace-policy-binding.k8s.jpbetz.github.io' denied request: environment not one of the required values: development, test, production

# Next, let's require labels for all our namespaces. We will require both a environment and owner label.
$ kubectl apply -f required-labels.yaml
$ kubectl apply -f ns-missing-labels.yaml
The namespaces "example-invalid-environment" is invalid: : ValidatingAdmissionPolicy 'required-labels.example.com' with binding 'required-labels-binding.example.com' denied request: Namespace missing required labels: environment

# Step 3: Limit developers to one sandbox each, and let's require it contain their username, e.g. "<username>-sandbox".
$ kubectl apply -f validate-developer-sandbox.yaml
$ kubectl apply -f jpbetz-rbac.yaml
$ kubectl apply -f ns-jpbetz-sandbox-invalid.yaml --as jpbetz
The namespaces "jpbetz-sandbox" is invalid: : ValidatingAdmissionPolicy 'developer-sandbox.example.com' with binding 'developer-sandbox-binding.example.com' denied request: Owner must be your username for a sandbox namespace. Expected: system:admin, got: jpbetzx
$ kubectl apply -f ns-jpbetz-sandbox-invalid2.yaml --as jpbetz
The namespaces "my-sandbox" is invalid: : ValidatingAdmissionPolicy 'developer-sandbox.example.com' with binding 'developer-sandbox-binding.example.com' denied request: Namespace name must be <username>-sandbox for a sandbox namespace. Expected: jpbetz-sandbox, got: my-sandbox
$ kubectl apply -f ns-jpbetz-sandbox-valid.yaml --as jpbetz

# Last, let's limit who can create production namespaces
# We'll take advantage of RBAC extensibility to create a synthetic resource kind to represent production namespaces
$ kubectl apply -f field-level-access-control.yaml
$ kubectl apply -f ns-production-invalid.yaml --as jpbetz
Error from server (Forbidden): error when deleting "ns-production-invalid.yaml": namespaces "not-allowed-for-user" is forbidden: ValidatingAdmissionPolicy 'production-namespaces.example.com' with binding 'production-namespaces-binding.example.com' denied request: failed expression: authorizer.group('authz.example.com').resource('production-namespaces').check('manage-prod-namespaces').allowed()

$ kubectl apply -f production-namespace-role.yaml
$ kubectl apply -f ns-production-valid.yaml --as alice
namespace/web-ui created

## CRDs...

# Imagine that we want to generalize what we just did with labels.
$ kubectl apply -f label-settings-crd.yaml
$ kubectl apply -f label-settings.yaml
$ kubectl apply -f required-labels-with-params.yaml
$ kubectl apply -f label-settings-invalid.yaml
The LabelSetting "label-setting.example.com" is invalid: requiredLabels[0].allowedValues: Invalid value: "array": Values may be added to allowedValues but not removed

$ kubectl delete validatingadmissionpolicy env-label.example.com
$ kubectl delete validatingadmissionpolicybinding env-label-binding.example.com
$ kubectl apply -f ns-invalid-environment.yaml
The namespaces "example-invalid-environment" is invalid: : ValidatingAdmissionPolicy 'required-labels.example.com' with binding 'required-labels-binding.example.com' denied request: label 'environment' must be one of: [developer-sandbox, test, production]

## Pods...

# We've focused a lot on labels
# But we're not limited to labels, we can do the same with any fields of any API
# resources (pod, deployments, custom resources, ...)

$ kubectl apply -f require-liveness-probe.yaml
$ kubectl apply -f invalid-deployment.yaml
The deployments "nginx-deployment" is invalid: : ValidatingAdmissionPolicy 'require-liveness-probe.example.com' with binding 'require-liveness-probe-binding.example.com' denied request: All containers must have a livenessProbe

$ kubectl apply -f require-container-registry.yaml

$ kubectl apply -f invalid-deployment2.yaml
The deployments "nginx-deployment" is invalid: : ValidatingAdmissionPolicy 'require-container-registry.example.com' with binding 'require-container-registry-binding.example.com' denied request: not all images are from the the required container registry: my-registry.io

# Warnings and audit annotations
$ kubectl apply -f require-container-registry-warn-audit.yaml
$ kubectl apply -f invalid-deployment2.yaml
Warning: Validation failed for ValidatingAdmissionPolicy 'require-container-registry.example.com' with binding 'require-container-registry-binding.example.com': not all images are from the the required container registry: my-registry.io
deployment.apps/nginx-deployment created
# See audit.json

## Ban use of Deprecated gitRepo volumes
$ kubectl apply -f ban-deprecated-gitrepo.yaml
$ kubectl apply -f pod-with-gitrepo.yaml
Warning: spec.volumes[0].gitRepo: deprecated in v1.11
The pods "server" is invalid: : ValidatingAdmissionPolicy 'ban-deprecated-gitrepo-volumes.example.com' with binding 'ban-deprecated-gitrepo-volumes-binding.example.com' denied request: Pod must not use deprecated .spec.volumes[].gitRepo field
```

What do I want to cover better?

- podspec-y things
- relationship with policy engines
- field level authz
- landmines: ratcheting

# Refs

- https://learnk8s.io/production-best-practices
- https://blog.openpolicyagent.org/securing-the-kubernetes-api-with-open-policy-agent-ce93af0552c3#3c6e
- https://spot.io/resources/kubernetes-autoscaling/5-kubernetes-deployment-strategies-roll-out-like-the-pros/?utm_campaign=spot.io-ps-kubernatics&utm_term=&utm_source=adwords&utm_medium=ppc&hsa_ver=3&hsa_kw=&hsa_cam=14123334381&hsa_tgt=dsa-1510472120825&hsa_acc=8916801654&hsa_mt=&hsa_net=adwords&hsa_ad=567046495853&hsa_src=g&hsa_grp=129315742943&gclid=CjwKCAjwzuqgBhAcEiwAdj5dRlDMzlL73WdiEC7jOYWkJ0q0ZmoG8bpORWfqO4f9NeuKCL_mJ-LNFRoCmoIQAvD_BwE

### 

```
cat audit.json | jq '.annotations."validation.policy.admission.k8s.io/validation_failure" | fromjson' > auditannotation.json
```
