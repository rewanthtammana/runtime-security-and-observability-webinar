# runtime-security-and-observability-workshop

## About me

https://rewanthtammana.com/

## Notes

### Install tetragon

```bash
helm repo add cilium https://helm.cilium.io
helm repo update
helm install tetragon cilium/tetragon -n kube-system
kubectl rollout status -n kube-system ds/tetragon -w
```

### Install tetra cli

```bash
GOOS=$(go env GOOS)
GOARCH=$(go env GOARCH)
curl -L --remote-name-all https://github.com/cilium/tetragon/releases/latest/download/tetra-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}
sha256sum --check tetra-${GOOS}-${GOARCH}.tar.gz.sha256sum
sudo tar -C /usr/local/bin -xzvf tetra-${GOOS}-${GOARCH}.tar.gz
rm tetra-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}
```

### Basic test

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=tetragon -c export-stdout -f | tetra getevents -o compact
```

```bash
kubectl run nginx --image nginx
```

### List capabilities

Edit the tetragon configuration & set `enable-process-cred` to `true`.

```bash
kubectl edit cm -n kube-system tetragon-config
kubectl rollout restart -n kube-system ds/tetragon
```

```bash
echo "apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: busybox
    command: [ "sleep", "1d" ]
    securityContext:
      privileged: true" | kubectl apply -f-
```

### Network observability

See the list of all connections that happen across cluster. There are some cases where domains are compromised & they will be pointed to a rogue IP. It's called DNS hijacking.

```bash
kubectl exec -it busybox sh
wget -qO- google.com
```

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/tetragon/main/examples/tracingpolicy/tcp-connect.yaml
```

```bash
kubectl exec -it busybox sh
wget -qO- google.com
```

### Actions available

https://github.com/cilium/tetragon/blob/main/docs/content/en/docs/reference/tracing-policy.md#actions-filter

Sigkill action, Override action, FollowFD action, UnfollowFD action, CopyFD action, GetUrl action, DnsLookup action, Post action

### Static pods demo

Create a pod in non-existant namespace via static pods, `/etc/kubernetes/manifests`. You cannot see that pod.

https://kubernetes.io/docs/concepts/security/api-server-bypass-risks/#static-pods

`crictl ps` & `kubectl get po -A`

How to prevent this attack?

Don't allow creation of files in `/etc/kubernetes/manifests`.

```bash
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "prevent-file-write"
spec:
  kprobes:
  # create entry for matching fd in a BPF map
  - call: "fd_install"
    syscall: false
    return: false
    args:
    - index: 0
      type: int
    - index: 1
      type: "file"
    selectors:
    - matchPIDs:
      - operator: NotIn
        followForks: true
        isNamespacePID: true 
        values:
        - 1
      matchArgs:
      - index: 1
        operator: "Prefix"
        values:
        - "/etc/kubernetes/manifests"
      matchActions:
      - action: FollowFD
        argFd: 0
        argName: 1
  # delete matching fd entry from BPF map
  - call: "__x64_sys_close"
    syscall: true
    args:
    - index: 0
      type: "int"
    selectors:
    - matchActions:
      - action: UnfollowFD
        argFd: 0
        argName: 0
  # read syscall is executed first
  - call: "__x64_sys_read"
    syscall: true
    args:
    - index: 0
      type: "fd"
    - index: 1
      type: "char_buf"
      returnCopy: true
    - index: 2
      type: "size_t"
  # look for write syscall in the matching directory & kill the operation with Sigkill
  - call: "__x64_sys_write"
    syscall: true
    args:
    - index: 0
      type: "fd"
    - index: 1
      type: "char_buf"
      sizeArgIndex: 3
    - index: 2
      type: "size_t"
    selectors: 
    - matchActions:
        - action: Sigkill
```

Create another policy to kill privileged pod. This can be done with kyverno too but it creates a webhook & it might break many things & it looks only for changes to kubernetes things. what if someone creates a docker container in privileged mode?


