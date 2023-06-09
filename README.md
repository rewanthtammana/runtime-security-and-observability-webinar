# runtime-security-and-observability-webinar

## About me

https://rewanthtammana.com/

## Notes

## Kubernetes API server bypass attack

Install pre-requisites & start a cluster

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
sudo apt update
sudo apt -y install docker.io vim net-tools bpfcc-tools linux-headers-$(uname -r)
docker pull kindest/node:v1.26.3
snap install kubectl helm --classic
kind create cluster
```


```bash
apiVersion: v1
kind: Pod
metadata:
  name: busybox-test
spec:
  containers:
  - name: busybox
    image: busybox
    command: [ wget, -qO-, 172.30.1.2:8000/privileged ]
    securityContext:
      privileged: true
```

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

```bash
kubectl edit cm -n kube-system tetragon-config
```

Edit the tetragon configuration & set `enable-process-cred` to `true`.


```bash
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

### Tracing policy for write functions

```bash
kubectl apply -f https://github.com/cilium/tetragon/raw/main/examples/tracingpolicy/write.yaml
```

### Actions available

https://github.com/cilium/tetragon/blob/main/docs/content/en/docs/reference/tracing-policy.md#actions-filter

Sigkill action, Override action, FollowFD action, UnfollowFD action, CopyFD action, GetUrl action, DnsLookup action, Post action

### Static pods demo

Create a pod in non-existant namespace via static pods, `/etc/kubernetes/manifests`. You cannot see that pod.

https://kubernetes.io/docs/concepts/security/api-server-bypass-risks/#static-pods

`crictl ps` & `kubectl get po -A`

How to prevent this attack?

Don't allow creation of files in `/etc/kubernetes/manifests`. For example, let's test with `/tmp/forbidden` directory.

```bash
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "deny-tmp-forbidden-write"
spec:
  kprobes:
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
        - "/tmp/forbidden/"
      matchActions:
      - action: FollowFD
        argFd: 0
        argName: 1
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

### Bypass above check

The list of syscalls are triggered when `vim` tries to make changes to the system. If we use some other method like, `echo abcd > /tmp/forbidden/hello`. It will bypass the above validation.

### List of syscalls made by process

Use strace. For ex,

```bash
strace ls
```

## References

* https://github.com/cilium/tetragon
* https://isovalent.com/blog
* https://b-nova.com/en/home/content/strengthen-your-system-with-tetragons-ebpf-based-security-observability-and-runtime-enforcement-capabilities

