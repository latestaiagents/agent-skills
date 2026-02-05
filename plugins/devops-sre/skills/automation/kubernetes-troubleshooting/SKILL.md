---
name: kubernetes-troubleshooting
description: |
  Diagnose and fix common Kubernetes issues with systematic debugging approaches.
  Use this skill when troubleshooting K8s clusters, pods not starting, deployments failing, or networking issues.
  Activate when: kubernetes, k8s, pod, deployment, kubectl, container, crashloopbackoff, imagepullbackoff,
  pending pods, kubernetes networking, service not working, ingress issues.
---

# Kubernetes Troubleshooting

**Systematic approaches to diagnose and fix common Kubernetes issues.**

## Troubleshooting Framework

```
1. What's the symptom? (pod not starting, service unreachable, etc.)
2. Where's the problem? (pod, service, ingress, node, cluster)
3. What do the events say?
4. What do the logs say?
5. What changed recently?
```

## Pod Issues

### Pod Status Quick Reference

| Status | Meaning | First Check |
|--------|---------|-------------|
| **Pending** | Can't be scheduled | `kubectl describe pod` |
| **ContainerCreating** | Image pulling or volume mounting | Events, `kubectl get events` |
| **CrashLoopBackOff** | Container crashes repeatedly | `kubectl logs --previous` |
| **ImagePullBackOff** | Can't pull container image | Image name, credentials |
| **Error** | Container exited with error | `kubectl logs` |
| **OOMKilled** | Out of memory | Increase memory limits |
| **Evicted** | Node under pressure | Node resources, pod priority |

### Debugging Commands

```bash
# Get pod status
kubectl get pod <pod-name> -o wide

# Describe pod (events, conditions)
kubectl describe pod <pod-name>

# Get logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>  # specific container
kubectl logs <pod-name> --previous       # previous crash

# Execute into pod
kubectl exec -it <pod-name> -- /bin/sh

# Get all events sorted by time
kubectl get events --sort-by='.lastTimestamp'
```

### CrashLoopBackOff

```
Symptoms: Pod restarts repeatedly
Common Causes:
├─ Application error on startup
├─ Missing config/secrets
├─ Liveness probe failing too soon
├─ Resource limits too low
└─ Dependency not ready

Debug Steps:
1. kubectl logs <pod> --previous
2. kubectl describe pod <pod>  # check events
3. Check liveness probe configuration
4. Check resource limits
5. Verify ConfigMaps/Secrets exist
```

### ImagePullBackOff

```
Symptoms: Container image can't be pulled
Common Causes:
├─ Image doesn't exist
├─ Wrong image name/tag
├─ Private registry, missing credentials
├─ Registry rate limiting
└─ Network issues

Debug Steps:
1. Verify image name: kubectl describe pod <pod>
2. Try pulling manually: docker pull <image>
3. Check imagePullSecrets in pod spec
4. Verify secret exists: kubectl get secret <secret-name>
5. Check registry status
```

### Pending Pods

```
Symptoms: Pod stuck in Pending state
Common Causes:
├─ Insufficient resources (CPU/memory)
├─ No nodes match nodeSelector/affinity
├─ PVC can't be bound
├─ Taint with no toleration
└─ ResourceQuota exceeded

Debug Steps:
1. kubectl describe pod <pod>  # check Events
2. kubectl get nodes -o wide   # check node capacity
3. kubectl describe node <node> # check allocatable
4. kubectl get pvc              # check volume claims
5. kubectl get resourcequota    # check quotas
```

## Service & Networking Issues

### Service Not Working

```bash
# Check service exists and has endpoints
kubectl get svc <service>
kubectl get endpoints <service>

# If no endpoints, check selector matches pods
kubectl get pods -l <selector-from-service>

# Test from inside cluster
kubectl run test --rm -it --image=busybox -- wget -qO- <service>:<port>

# Check DNS resolution
kubectl run test --rm -it --image=busybox -- nslookup <service>
```

### Debugging Checklist

```
□ Service exists and has correct port
□ Endpoints exist (pods are selected)
□ Pod selector labels match
□ Pods are Running and Ready
□ Container is listening on correct port
□ NetworkPolicy isn't blocking traffic
□ DNS resolves correctly
```

### Ingress Issues

```bash
# Check ingress configuration
kubectl describe ingress <ingress-name>

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Verify backend service
kubectl get svc <backend-service>

# Check TLS secret
kubectl get secret <tls-secret>
```

## Deployment Issues

### Deployment Not Rolling Out

```bash
# Check rollout status
kubectl rollout status deployment/<name>

# Check deployment events
kubectl describe deployment <name>

# Check replicaset
kubectl get rs -l app=<name>
kubectl describe rs <replicaset-name>

# Rollback if needed
kubectl rollout undo deployment/<name>
```

### Common Deployment Problems

```
Symptom: New pods not creating
Check:
├─ ResourceQuota limits
├─ PodDisruptionBudget blocking
├─ Node capacity

Symptom: Old pods not terminating
Check:
├─ terminationGracePeriodSeconds
├─ PreStop hooks stuck
├─ Finalizers blocking deletion

Symptom: Rollout stuck
Check:
├─ maxUnavailable settings
├─ Readiness probe never passes
├─ PVC can't be detached
```

## Node Issues

### Node Not Ready

```bash
# Check node status
kubectl get nodes
kubectl describe node <node>

# Check node conditions
kubectl get node <node> -o jsonpath='{.status.conditions[*].type}'

# Check kubelet logs (on node)
journalctl -u kubelet -f

# Drain node if needed
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

### Resource Pressure

```bash
# Check node resources
kubectl top nodes

# Check which pods are using resources
kubectl top pods --all-namespaces

# Find pods on specific node
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node>
```

## Quick Diagnostic Commands

```bash
# Overall cluster health
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running
kubectl get events --sort-by='.lastTimestamp' | tail -20

# Specific namespace health
kubectl get all -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Resource usage
kubectl top nodes
kubectl top pods -n <namespace>

# Network debugging pod
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash
```

## Systematic Debug Template

```markdown
## Issue: [Brief description]

### Symptom
[What's happening]

### Affected Resources
- Namespace:
- Deployment/Pod:
- Service:

### Investigation

#### Step 1: Check Status
```
kubectl get pod <pod>
kubectl describe pod <pod>
```
Findings: [...]

#### Step 2: Check Logs
```
kubectl logs <pod>
```
Findings: [...]

#### Step 3: Check Events
```
kubectl get events --sort-by='.lastTimestamp'
```
Findings: [...]

### Root Cause
[What caused the issue]

### Resolution
[What fixed it]

### Prevention
[How to prevent recurrence]
```

## Emergency Procedures

### Force Delete Stuck Pod

```bash
# Only use when pod is truly stuck
kubectl delete pod <pod> --grace-period=0 --force
```

### Emergency Rollback

```bash
# Immediate rollback
kubectl rollout undo deployment/<name>

# Rollback to specific revision
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=<n>
```

### Scale Down Quickly

```bash
# Scale to zero
kubectl scale deployment/<name> --replicas=0

# Scale back up
kubectl scale deployment/<name> --replicas=3
```
