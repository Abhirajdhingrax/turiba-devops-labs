# Lab 7 · Run it on a cluster

Cluster: local kind cluster (context kind-kind). Image: ghcr.io/martinsturiba/turiba-devops-labs/api:0.2.0, the course fallback, because I have no 0.2.0 release of my own.

## 1. Deploy

Prediction: 3 Pods, with names starting with api- (the Deployment name, then the ReplicaSet hash, then a random suffix). Result: correct.

    NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/api   3/3     3            3           31h
    
    NAME                             DESIRED   CURRENT   READY   AGE
    replicaset.apps/api-7ff476cc44   3         3         3       31h
    
    NAME                       READY   STATUS    RESTARTS        AGE
    pod/api-7ff476cc44-8944l   1/1     Running   1 (3m40s ago)   31h
    pod/api-7ff476cc44-cbdhc   1/1     Running   1 (3m40s ago)   31h
    pod/api-7ff476cc44-mrqrj   1/1     Running   1 (3m40s ago)   31h

GET / through kubectl port-forward service/api 8080:80:

    {"name":"course-api","version":"0.1.0"}
## 2. Heal and scale

Prediction: still 3 Pods two seconds after deleting one, because the ReplicaSet makes a new one at once; the new Pod has a new name and an age of a few seconds. Result: correct, the new Pod was 3 seconds old.

    Deleting pod/api-7ff476cc44-8944l
    NAME                   READY   STATUS    RESTARTS        AGE
    api-7ff476cc44-cbdhc   1/1     Running   1 (3m47s ago)   31h
    api-7ff476cc44-mrqrj   1/1     Running   1 (3m47s ago)   31h
    api-7ff476cc44-pk2jm   1/1     Running   0               3s

After changing replicas to 5 and applying (not committed):

    NAME                   READY   STATUS    RESTARTS        AGE
    api-7ff476cc44-52r8c   1/1     Running   0               8s
    api-7ff476cc44-cbdhc   1/1     Running   1 (3m56s ago)   31h
    api-7ff476cc44-mrqrj   1/1     Running   1 (3m56s ago)   31h
    api-7ff476cc44-pk2jm   1/1     Running   0               12s
    api-7ff476cc44-xqh4q   1/1     Running   0               8s

After git restore k8s/ and apply again, back to 3 and git diff is empty:

    NAME                   READY   STATUS    RESTARTS       AGE
    api-7ff476cc44-cbdhc   1/1     Running   1 (4m5s ago)   31h
    api-7ff476cc44-mrqrj   1/1     Running   1 (4m5s ago)   31h
    api-7ff476cc44-pk2jm   1/1     Running   0              21s

## 3. Roll and roll back

3a: I ran out of time for a separate tag-change PR, so I did not roll to a newer image.

3b: In my working copy only, I changed the tag to 9.9.9, which does not exist, and applied it. Prediction: all 3 old Pods keep running, because the default rolling update allows 0 unavailable Pods and the new Pod never becomes ready. Rollout status timed out and the new Pod stayed in ErrImagePull/ImagePullBackOff while the old Pods kept running:

    Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
    error: timed out waiting for the condition
    NAME                   READY   STATUS         RESTARTS        AGE
    api-66bfb5cf85-4qxg7   0/1     ErrImagePull   0               30s
    api-7ff476cc44-cbdhc   1/1     Running        1 (5m16s ago)   31h
    api-7ff476cc44-mrqrj   1/1     Running        1 (5m16s ago)   31h
    api-7ff476cc44-pk2jm   1/1     Running        0               92s

kubectl -n course rollout undo deployment/api:

    Warning: resource deployments/api was previously managed with 'kubectl apply'. Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration annotation, which may cause unexpected behavior on future 'kubectl apply' operations. Consider using 'kubectl apply' with your previous configuration file instead.
    deployment.apps/api rolled back

What undo means: it puts the cluster back to the previous ReplicaSet, but it does not change my file. My working copy still said 9.9.9, so Git and the cluster disagreed. I ran git restore k8s/ and kubectl apply -f k8s/ so they agree again:

    deployment "api" successfully rolled out

## 4. What broke and how I found it

Prediction: the image would fail first. I applied the teammate's branch (chore/tidy-manifests, PR #6) and used the debug loop.

Fault 1, image tag: kubectl -n course get pods showed the new Pod stuck in ErrImagePull/ImagePullBackOff. The tag was 0.2.O with the letter O instead of a zero. Fix commit: fix(k8s): correct image tag 0.2.O to 0.2.0.

    NAME                   READY   STATUS             RESTARTS        AGE
    api-7ff476cc44-cbdhc   1/1     Running            1 (5m41s ago)   31h
    api-7ff476cc44-mrqrj   1/1     Running            1 (5m41s ago)   31h
    api-7ff476cc44-pk2jm   1/1     Running            0               117s
    api-85fbbc455c-8rzkh   0/1     ImagePullBackOff   0               21s

Fault 2, Service selector: kubectl -n course get endpointslices -l kubernetes.io/service-name=api showed no endpoints, because the selector became app.kubernetes.io/name: api while the Pods are labelled app: api. Fix commit: fix(k8s): make the service selector match the pod label app: api.

    NAME        ADDRESSTYPE   PORTS     ENDPOINTS   AGE
    api-pfkv6   IPv4          <unset>   <unset>     31h

Fault 3, targetPort: in the PR's Files changed tab, targetPort was 8080, but the container listens on 5000 (containerPort 5000). Through port-forward that gives connection refused. Fix commit: fix(k8s): send service traffic to port 5000 where the API listens.

After the three fixes, kubectl apply -f k8s/ rolled out 3/3 and GET / answered through the port-forward:

    {"name":"course-api","version":"0.1.0"}
