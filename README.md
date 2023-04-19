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

