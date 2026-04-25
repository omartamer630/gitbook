---
cover: ../.gitbook/assets/Gemini_Generated_Image_thr806thr806thr8.png
coverY: 0
---

# Hands-On Task: Debugging a Failing Pod

### Task Overview <a href="#heading-task-overview" id="heading-task-overview"></a>

In this task, we intentionally break a Kubernetes Pod, observe how Kubernetes reacts, and then fix the issue **without deleting the Pod**.

#### Task Goals <a href="#heading-task-goals" id="heading-task-goals"></a>

* Create a Pod name `debug-pod`
* Force it into a failure state, we will use`ErrImagePull / ImagePullBackOff` failure
* Debug the failure using Kubernetes events.
* Fix the issue live.
* Verify the Pod reaches `Running`
* Capture and document every step you do.

***

### Step 1: Create the Pod with an Invalid Image <a href="#heading-step-1-create-the-pod-with-an-invalid-image" id="heading-step-1-create-the-pod-with-an-invalid-image"></a>

Create a Pod named `debug-pod` using an **incorrect container image name**.

The mistake is intentional.

Example idea (not mandatory how you created it):

* Image name: `ngimx:alpine` (typo)

After creation, check the Pod status:

```bash
kubectl get pods
```

#### Expected Result <a href="#heading-expected-result" id="heading-expected-result"></a>

```bash
debug-pod   0/1   ErrImagePull   0   <time>
```

After some retries, it transitions to:

```bash
ImagePullBackOff
```

***

### Step 2: Inspect the Pod and Identify the Failure <a href="#heading-step-2-inspect-the-pod-and-identify-the-failure" id="heading-step-2-inspect-the-pod-and-identify-the-failure"></a>

Describe the Pod to understand what went wrong:

```bash
kubectl describe pod debug-pod
```

In Pod Describe you will see Events part with ErrImagePull:

```bash
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  4m19s                  default-scheduler  Successfully assigned default/de-pod to kind-worker
  Normal   Pulling    3m40s (x3 over 4m19s)  kubelet            Pulling image "ngimx"
  Warning  Failed     3m39s (x3 over 4m18s)  kubelet            Failed to pull image "ngimx": failed to pull and unpack image "docker.io/library/ngimx:latest": failed to resolve reference "docker.io/library/ngimx:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     3m39s (x3 over 4m18s)  kubelet            Error: ErrImagePull
  Normal   BackOff    3m23s (x3 over 4m18s)  kubelet            Back-off pulling image "ngimx"
  Warning  Failed     3m23s (x3 over 4m18s)  kubelet            Error: ImagePullBackOff
```

#### What to Look For <a href="#heading-what-to-look-for" id="heading-what-to-look-for"></a>

In the **Events** section, you should see entries similar to:

* `Pulling image "ngimx:alpine"`
* `Failed to pull image`
* `ErrImagePull`
* `ImagePullBackOff`

This confirms:

* Kubernetes scheduling worked
* The failure happened at the **image pull stage**
* The issue is **external dependency failure**, not scheduling or YAML syntax

***

### Step 3: Fix the Pod Without Deleting It <a href="#heading-step-3-fix-the-pod-without-deleting-it" id="heading-step-3-fix-the-pod-without-deleting-it"></a>

Instead of deleting and recreating the Pod, edit it live:

```bash
kubectl edit pod debug-pod
```

Fix the image name:

```yaml
image: nginx:alpine
```

Save and exit.

***

### Step 4: Verify Pod Recovery <a href="#heading-step-4-verify-pod-recovery" id="heading-step-4-verify-pod-recovery"></a>

Check the Pod status again:

```bash
kubectl get pods
```

#### Expected Result <a href="#heading-expected-result-1" id="heading-expected-result-1"></a>

```
debug-pod   1/1   Running   0   <time>
```

Describe the Pod again:

```bash
kubectl describe pod debug-pod
```

The output should be like:

```basic
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  4m19s                  default-scheduler  Successfully assigned default/de-pod to kind-worker
  Normal   Pulling    3m40s (x3 over 4m19s)  kubelet            Pulling image "ngimx"
  Warning  Failed     3m39s (x3 over 4m18s)  kubelet            Failed to pull image "ngimx": failed to pull and unpack image "docker.io/library/ngimx:latest": failed to resolve reference "docker.io/library/ngimx:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     3m39s (x3 over 4m18s)  kubelet            Error: ErrImagePull
  Normal   BackOff    3m23s (x3 over 4m18s)  kubelet            Back-off pulling image "ngimx"
  Warning  Failed     3m23s (x3 over 4m18s)  kubelet            Error: ImagePullBackOff
  Normal   Pulling    3m22s                  kubelet            Pulling image "nginx"
  Normal   Pulled     2m51s                  kubelet            Successfully pulled image "nginx" in 31.548s (31.548s including waiting). Image size: 62870438 bytes.
  Normal   Created    2m51s                  kubelet            Created container de-pod
  Normal   Started    2m51s                  kubelet            Started container de-pod
```

#### Observations <a href="#heading-observations" id="heading-observations"></a>

* Container is now `Running`
* Image was pulled successfully
* Previous failure events are still visible

This is expected behavior.

***

### Step 5: Verify Pod YAML <a href="#heading-step-5-verify-pod-yaml" id="heading-step-5-verify-pod-yaml"></a>

Export the final Pod manifest:

```bash
kubectl get pod debug-pod -o yaml > debug-pod.yaml
```

Key things to notice in the YAML:

* Correct image name
* `restartPolicy: Always`
* `qosClass: BestEffort` (no resource requests or limits)

***

### Important Notes <a href="#heading-important-notes" id="heading-important-notes"></a>

#### **Why Old Events Still Exist** <a href="#heading-why-old-events-still-exist" id="heading-why-old-events-still-exist"></a>

If you run:

```bash
kubectl describe pod debug-pod
```

you may see the same events as in a previous `kubectl describe`.

This happens because Kubernetes:

* Does **not rewrite or delete old events** when a Pod is fixed.
* Treats events as **append-only** and **time-limited**.
* Keeps old `ErrImagePull` events visible even after the Pod recovers this is normal.

Kubernetes does **not retain events forever**. Each event has a **time-to-live (TTL)** and is deleted automatically after a certain period. By default, Kubernetes events expire **1 hour** after they are created.

#### Hidden Observation <a href="#heading-hidden-observation" id="heading-hidden-observation"></a>

The Pod is running with:

```bash
QoS Class: BestEffort
```

This means:

* No CPU or memory requests
* No resource guarantees
* First candidate for eviction under node pressure

***

### Final Result <a href="#heading-final-result" id="heading-final-result"></a>

By the end of this task, we:

* Forced a real Kubernetes failure
* Used events to identify the root cause
* Fixed the issue without recreating resources
* Observed actual Pod state transitions

***

### Reference <a href="#heading-reference" id="heading-reference"></a>

* [Kubernetes Documentation – Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
* [Kubernetes Documentation – Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
* [Kubernetes Documentation – Container Images](https://kubernetes.io/docs/concepts/containers/images/)
* [Kubernetes Documentation – Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
* [Kubernetes Documentation – Pod Restart Policy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)
* [Kubernetes Documentation – Resource Quality of Service](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) (For the next article)
