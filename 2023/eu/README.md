# Tricks for enforcing conventions for your Kubernetes cluster using only YAML

Have you ever operated a Kubernetes cluster for multiple developers? If you have, you probably realized quickly that things are going to be a lot smoother if you could just enforce some basic conventions. Maybe all your services have a well defined endpoint for the liveness probe but developers sometimes forget to set it up. Or maybe developers should always use a semantic version tag on their containers and avoid :latest. Or maybe there is a deprecated Kubernetes API field and you'd like to ensure it is never used in your cluster. In this talk we will run through a series of easy solutions to help enforce conventions using only YAML. You have a lot more control that you might realize. Learn from a Kubernetes contributor involved in the development of numerous extensibility features including CRDs, admission webhooks and admission policies. We will show you some handy tricks and leveraging new features like Validating Admission Policies alpha API introduced in 1.26.

# Examples

Admission policy stuff:

- [x] Liveness probe endpoint requirement
- [x] Semantically versioned containers
- [ ] Field level access control.
  - E.g  label X can only be changed with a specific permission
- [ ] Using fake authz verbs and resources.
  - E.g. "breakglass" verb
- [ ] Validating labels and metadata.
  - E.g. environment must be one of "test, staging, production"
- PoCo: K8sContainerLimits, K8sReplicaLimits, K8sRequiredProbes, K8sRequiredLabels, K8sRequiredAnnotations
- Sidecar example?
- [ ] "I want all my production images to be pulled from a trusted registry"

CRD Stuff:

- [ ] Making CRD fields immutable.
  - [ ] Show the simple required case


Core stuff:

- Set Resource Requests and Limits
- RBAC

# Outline

- Quick intro
- Image registry
- Namespace control
  - Basic RBAC
  - Label control: required labels, label value validation
  - Secondary Authz: permission for "production" environment label value
- CRD validation
  - immutability
  - state transitions (enum)
  - ratcheting

- Other examples
  - Require probes

- Safe rollout (actions)
- Extensibility (params, bindings)

- Closing
  - Comparison with policy engines
    - This is a single enforcement point
      - Provides building blocks for:
        - reporting via audit and audit backends
        - reuse via params, bindings and actions
      - But more comprehensive policy systems will need build on this


# Notes

- Policy engines exist and that they have nice bundles for things like pod security policies, compliance and such.
- But sometimes you just want to setup a simple rule for how your k8s cluster is supposed to be used.
- I won't cover basic RBAC today, but that is always a good first stop when setting up a multi-user cluster.

- Things are good, for a while.

- Then as multiple teams start to use the cluster, we see the need to start to "shape" the cluster.
  The first step is creating namespaces and organizing the namespaces with labels.
- Now you might organize things different than I show here, but let's just say, hypothetically, that our
  entire organization uses a single Kuberentes cluster.
- So we'll use labels to help organize namespaces. Let's say we're going to have a "environment" and "owner" label
  for each namespace.  The environment can be "development" or "test" or "production", the "owner" can be the name of
  an individual or team.
- We can use RBAC to control who creates namespaces, which is great.
- But there's still a lot of room for improvement.
- Why can't we require the labels always be set?  Why can't we control which values are allowed?
- Example: Require-labels, require-label-values
- That's a start.  For production, I'd really like to limit who can create namespaces to a smaller group of people.
- Example: production-namespace-role
- But I just invented the "production-control" RBAC verb. Right now it doesn't actually do anything
- But we can use it, to limit who can create and edit production namespaces
- Example: field-level-access-control

We can also do more typical policy things like:

- limiting what image registries may be used
- requiring all deployment containers have liveness probes
- requiring semver image versions

We can control the rollout of any of these changes by first using a ValidatingAdmissionPolicyBinding configured to only warn and audit
violations, and then, once we're ready, transition to enforcement.

- Example of a ValidatingAdmissionPolicyBinding

We might want to automate further by defining operators to perform important tasks. To do this we will need
CRDs. Which brings us to another 


# TODO

- Show using params for configurable things like the list of namespaces
- Run everything through a cluster, make sure it actually works
- Show image registry example first, then do namespaces
- Example where everyone is able to create a sandbox namespace for themself (a controller could set up LimitRange and ResourceQuota for all sanbox namespaces - Kyverno supports this)

# Refs

- https://learnk8s.io/production-best-practices
- https://blog.openpolicyagent.org/securing-the-kubernetes-api-with-open-policy-agent-ce93af0552c3#3c6e
- https://spot.io/resources/kubernetes-autoscaling/5-kubernetes-deployment-strategies-roll-out-like-the-pros/?utm_campaign=spot.io-ps-kubernatics&utm_term=&utm_source=adwords&utm_medium=ppc&hsa_ver=3&hsa_kw=&hsa_cam=14123334381&hsa_tgt=dsa-1510472120825&hsa_acc=8916801654&hsa_mt=&hsa_net=adwords&hsa_ad=567046495853&hsa_src=g&hsa_grp=129315742943&gclid=CjwKCAjwzuqgBhAcEiwAdj5dRlDMzlL73WdiEC7jOYWkJ0q0ZmoG8bpORWfqO4f9NeuKCL_mJ-LNFRoCmoIQAvD_BwE

## Commands

```sh
## Namespaces

# First let's imagine we want an environment label on our namespaces, and we want to control what values it can be set to.
$ k apply --server-side -f validate-environment-label.yaml --validate=false
$ k apply -f ns-invalid-environment.yaml
The namespaces "example-invalid-environment" is invalid: : ValidatingAdmissionPolicy 'production-namespace-policy.k8s.jpbetz.github.io' with binding 'production-namespace-policy-binding.k8s.jpbetz.github.io' denied request: environment not one of the required values: development, test, production

# Next, let's require labels for all our namespaces.  We will require both a environment and owner label.
$ k apply --server-side -f required-labels.yaml --validate=false
$ k apply -f ns-missing-labels.yaml
The namespaces "example-invalid-environment" is invalid: : ValidatingAdmissionPolicy 'required-labels.example.com' with binding 'required-labels-binding.example.com' denied request: Namespace missing required labels: environment

# For the "developer-sandbox" namespaces, let's also limit developers to one sandbox each, and let's require it contain their username, e.g. "<username>-sandbox".
$ k apply --server-side -f validate-developer-sandbox.yaml --validate=false
$ k apply -f jpbetz-rbac.yaml
$ k apply -f ns-jpbetz-sandbox-valid.yaml --as jpbetz

# Last, let's limit who can create production namespaces
# We'll take advantage of RBAC extensibility to create a synthetic resource kind to represent production namespaces
$ k apply --server-side -f field-level-access-control.yaml --validate=false
$ k apply -f ns-production-invalid.yaml --as jpbetz
Error from server (Forbidden): error when deleting "ns-production-invalid.yaml": namespaces "not-allowed-for-user" is forbidden: ValidatingAdmissionPolicy 'production-namespaces.example.com' with binding 'production-namespaces-binding.example.com' denied request: failed expression: authorizer.group('authz.example.com').resource('production-namespaces').check('manage-prod-namespaces').allowed()

$ k apply -f production-namespace-role.yaml
$ k apply -f ns-production-valid.yaml --as alice
namespace/web-ui created

## CRDs

# Imagine that we want to generalize what we just did with labels.
$ k apply --server-side -f label-settings-crd.yaml
$ k apply -f label-settings.yaml
$ k apply --server-side -f required-labels-with-params.yaml
$ k apply -f label-settings-invalid.yaml
The LabelSetting "label-setting.example.com" is invalid: requiredLabels[0].allowedValues: Invalid value: "array": Values may be added to allowedValues but not removed

$ k delete validatingadmissionpolicy env-label.example.com
$ k delete validatingadmissionpolicybinding env-label-binding.example.com
$ k apply -f ns-invalid-environment.yaml
The namespaces "example-invalid-environment" is invalid: : ValidatingAdmissionPolicy 'required-labels.example.com' with binding 'required-labels-binding.example.com' denied request: label 'environment' must be one of: [developer-sandbox, test, production]

## Pods

$ k apply -f require-liveness-probe.yaml
$ k apply -f invalid-deployment.yaml
The deployments "nginx-deployment" is invalid: : ValidatingAdmissionPolicy 'require-liveness-probe.example.com' with binding 'require-liveness-probe-binding.example.com' denied request: All containers must have a livenessProbe

$ k apply -f require-container-registry.yaml

$ k apply -f invalid-deployment2.yaml
The deployments "nginx-deployment" is invalid: : ValidatingAdmissionPolicy 'require-container-registry.example.com' with binding 'require-container-registry-binding.example.com' denied request: not all images are from the the required container registry: my-registry.io

## Breakglass - so we have some way to bypass our restrictions, if we ever need to

# Image we'd like to have some way to bypass any of our rules
$ 

```
