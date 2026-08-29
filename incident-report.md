# Kubernetes Incident Report

## Student Scope

- IAM username: `charles`
- Resource prefix: `w12-charles-`
- Namespace: `dev`

## Incident 1 — Auth Workload

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

## Incident 2 — Wallet Service Connectivity

Investigation pending.

## Incident 3 — Notifications Restarts

Investigation pending.

## Incident 4 — Audit Scheduling

Investigation pending.

## Incident 5 — Transfer Image

Investigation pending.

## Final Verification

Verification pending.