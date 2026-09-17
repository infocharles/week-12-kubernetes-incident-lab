# Kubernetes Incident Report

## Student Scope

- IAM username: `charles`
- Resource prefix: `w12-charles-`
- Namespace: `dev`

## Incident 1 — Auth Workload

### Investigation commands

- `kubectl describe pod w12-charles-auth-674f99fcf5-ctzwk -n dev`
- `kubectl get secrets -n dev`
- `kubectl get deployment w12-charles-auth -n dev -o yaml`

### Symptoms and evidence

The Auth Pod was not starting and remained in `CreateContainerConfigError`. The Pod Events reported:

`Error: secret "ledgeway-env-missing" not found`

A read-only check of the available Secrets in the `dev` namespace showed that `ledgeway-env` existed, while `ledgeway-env-missing` did not.

### Root cause

The Auth Deployment referenced a non-existent, non-optional Secret named `ledgeway-env-missing` through `envFrom`. Kubernetes could not finish creating the container because the required configuration dependency was unavailable.

### Correction

I saved the live Auth Deployment as `manifests/auth-deployment.yaml` and changed the Secret reference to `ledgeway-env`.

During rollout, the replacement Pod could not schedule because both cluster nodes had reached their pod limit. I changed the one-replica RollingUpdate strategy to `maxSurge: 0` and `maxUnavailable: 1`, allowing Kubernetes to replace the already-unavailable Pod without requiring an additional cluster slot. I applied only `manifests/auth-deployment.yaml`.

### Verification

`kubectl rollout status deployment/w12-charles-auth -n dev` completed successfully. The Deployment became `1/1` available, and the replacement Auth Pod became `1/1 Running` with zero restarts.

Evidence:

- `evidence/01-auth-before.png`
- `evidence/01-auth-after.png`

### Prevention recommendation

Validate all Secret and ConfigMap references against the target namespace before deployment. A server-side dry run and deployment checklist should confirm that every required configuration dependency exists.

## Incident 2 — Wallet Service Connectivity

### Investigation commands

- `kubectl get pods -n dev -l cloudpros.io/student=charles --show-labels`
- `kubectl get service w12-charles-wallet -n dev -o jsonpath='{.spec.selector}{"\n"}'`
- `kubectl get endpoints w12-charles-wallet -n dev`

### Symptoms and evidence

The Wallet Pod was healthy and `1/1 Running`, but the Wallet Service showed `<none>` under Endpoints.

### Root cause

A Kubernetes Service finds its backend Pods by comparing its selector with the labels assigned to Pods. The Wallet Pod had the label `app=w12-charles-wallet`, but the Service was selecting `app=w12-charles-wallet-backend`. Because these values did not match, the Service could not select the healthy Wallet Pod.

### Correction

I saved the live Wallet Service as `manifests/wallet-service.yaml` and changed its selector to `app=w12-charles-wallet`. I validated the manifest with a server-side dry run and applied only `manifests/wallet-service.yaml`.

### Verification

`kubectl get endpoints w12-charles-wallet -n dev` changed from `<none>` to a Pod IP and port, confirming that the Service selected the Wallet Pod successfully. The Wallet Pod remained `1/1 Running`.

Evidence:

- `evidence/03-wallet-before.png`
- `evidence/04-wallet-after.png`

### Prevention recommendation

Use consistent application labels and Service selectors defined from the same configuration source. Verify Service endpoints after deployment so selector mismatches are detected immediately.

## Incident 3 — Notifications Restarts

### Investigation commands

- `kubectl get pods -n dev -l cloudpros.io/student=charles`
- `kubectl logs w12-charles-notifications-7c4fd7457c-7hz6l -n dev --previous --tail=50`
- `kubectl describe pod w12-charles-notifications-7c4fd7457c-7hz6l -n dev`
- `kubectl get deployment w12-charles-notifications -n dev -o yaml`

### Symptoms and evidence

The Notifications Pod was in `CrashLoopBackOff`, was not Ready, and had restarted 693 times when investigated. The previous container logs showed that Nginx started successfully, but requests from `kube-probe` to `/not-a-real-endpoint` returned HTTP `404`.

The Pod Events reported that the liveness probe failed with status code `404` and that kubelet was killing and restarting the container.

### Root cause

A container can start successfully but still be restarted when its liveness probe continually fails. The Notifications application was running, but its liveness probe checked the invalid path `/not-a-real-endpoint`. After three failed checks, kubelet treated the container as unhealthy and restarted it.

### Correction

I saved the live Notifications Deployment as `manifests/notifications-deployment.yaml` and changed the liveness-probe path to `/`.

During the rollout, the corrected replacement Pod could not schedule because both cluster nodes had reached their Pod limit. I changed the one-replica RollingUpdate strategy to `maxSurge: 0` and `maxUnavailable: 1`, allowing Kubernetes to replace the unavailable old Pod without requiring an additional cluster slot. I applied only `manifests/notifications-deployment.yaml`.

### Verification

`kubectl rollout status deployment/w12-charles-notifications -n dev --timeout=60s` completed successfully. Repeated runs of `kubectl get pods -n dev -l app=w12-charles-notifications` showed the replacement Pod remained `1/1 Running` with zero restarts.

Evidence:

- `evidence/05-notifications-before.png`
- `evidence/06-notifications-after.png`

### Prevention recommendation

Use a health-check endpoint that is guaranteed to return a successful response, and test probe paths before deployment. Monitor rollout status and restart counts after every probe change.

## Incident 4 — Audit Scheduling

### Investigation commands

- `kubectl get pods -n dev -l cloudpros.io/student=charles`
- `kubectl describe pod w12-charles-audit-6bfc4fbcf5-2hqlf -n dev`
- `kubectl get deployment w12-charles-audit -n dev -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'`
- `kubectl get nodes -o custom-columns='NAME:.metadata.name,ALLOCATABLE_MEMORY:.status.allocatable.memory,ALLOCATABLE_PODS:.status.allocatable.pods'`

### Symptoms and evidence

The Audit Pod remained `Pending`. Its scheduler Events initially reported `Insufficient memory` and `Too many pods`.

The Audit container requested and limited itself to `20Gi` of memory, while each cluster node had only approximately `1.4Gi` of allocatable memory.

### Root cause

A resource request is the amount of CPU or memory Kubernetes reserves when deciding where a Pod can be scheduled. A resource limit is the maximum amount the container is permitted to use while running.

The Audit Pod requested `20Gi` of memory, which was greater than the allocatable memory on either node. Therefore, the scheduler could not place it.

### Correction

I saved the live Audit Deployment as `manifests/audit-deployment.yaml`. Because it uses the same Nginx image as the healthy Notifications workload, I changed its memory request to `16Mi` and limit to `32Mi`, with a CPU request of `5m` and limit of `25m`.

I also set the one-replica RollingUpdate strategy to `maxSurge: 0` and `maxUnavailable: 1`, preventing the Deployment from requiring an additional Pod slot during replacement. I applied only `manifests/audit-deployment.yaml`.

After the memory correction, the `Insufficient memory` error disappeared, but the shared cluster temporarily remained at its Pod limit. The instructor increased the cluster size, allowing the corrected Audit Pod to schedule.

### Verification

`kubectl rollout status deployment/w12-charles-audit -n dev --timeout=60s` completed successfully. `kubectl get pods -n dev -l app=w12-charles-audit -o wide` showed the corrected Pod scheduled onto a node and `1/1 Running` with zero restarts.

Evidence:

- `evidence/07-audit-before.png`
- `evidence/08-audit-capacity-blocker.png`
- `evidence/09-audit-after.png`

### Prevention recommendation

Define realistic resource requests based on observed workload usage and cluster capacity. Use namespace LimitRanges or policy checks to reject excessive requests such as `20Gi` before they reach the cluster.

## Incident 5 — Transfer Image

### Investigation commands

- `kubectl get pods -n dev -l cloudpros.io/student=charles`
- `kubectl describe pod w12-charles-transfer-795bdb4f6-6gptj -n dev`
- `kubectl get deployment w12-charles-transfer -n dev -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'`
- `kubectl get deployment w12-charles-notifications -n dev -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'`

### Symptoms and evidence

The Transfer Pod remained in `ImagePullBackOff`. Its Events showed that kubelet repeatedly failed and backed off while attempting to pull `public.ecr.aws/docker/library/nginx:week12-intentionally-missing`.

### Root cause

Kubernetes must retrieve a container image before it can create and start the container. The Transfer Deployment referenced an image tag that did not exist in the registry, so the node could not download the image and the Pod could not start.

### Correction

I saved the live Transfer Deployment as `manifests/transfer-deployment.yaml`. I replaced the unavailable image with the known-working image `public.ecr.aws/docker/library/nginx:1.27-alpine`, validated the manifest with a server-side dry run, and applied only `manifests/transfer-deployment.yaml`.

### Verification

`kubectl rollout status deployment/w12-charles-transfer -n dev --timeout=60s` completed successfully. `kubectl get pods -n dev -l app=w12-charles-transfer` showed the replacement Pod `1/1 Running` with zero restarts. The live Deployment showed the corrected image `public.ecr.aws/docker/library/nginx:1.27-alpine`.

Evidence:

- `evidence/10-transfer-before.png`
- `evidence/11-transfer-after.png`

### Prevention recommendation

Use approved, versioned image tags and verify that each image exists in its registry before deployment. CI should validate or pull the configured image before allowing the manifest to be merged.

## Final Verification

Commands used:

- `kubectl get pods -n dev -l cloudpros.io/student=charles`
- `kubectl get endpoints w12-charles-wallet -n dev`

All five assigned Pods—Auth, Wallet, Notifications, Audit, and Transfer—were `1/1 Running` with zero restarts. The Wallet Service had an active backend endpoint.

Evidence:

- `evidence/12-final-verification.png`