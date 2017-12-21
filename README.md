# freebsd-sidecar
A very experimental way to run FreeBSD as a Kubernetes workload 

## The Dockerfile

```bash
cat < EOF >> Dockerfile
FROM alpine
RUN apk update && apk add qemu-system-x86_64 libvirt qemu-img bash

COPY freebsd.img /base_image/freebsd.img
CMD ["/usr/bin/qemu-system-x86_64", "-curses", "/base_image/freebsd.img"]
EOF
```

## Deploying on Kubernetes

1. Create a deployment file

```bash
cat < EOF >> deployment.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: freebsd-sidecar
spec:
  template:
    metadata:
      labels:
        app: freebsd
    spec:
      volumes:
        - name: "kvm"
          hostPath:
            path: "/dev/kvm"
      containers:
      - name: freebsd
        image: dcasati/inceptionbsd
        stdin: true
        tty: true
        volumeMounts:
          - mountPath: "/dev/kvm"
            name: "kvm"
        ports:
        - containerPort: 22
 ```
2. Deploy

```bash
$ kubectl create -f deployment.yml
deployment "freebsd-sidecar" created
```

3. Check the deployment

```bash
$ kubectl get pod -l app=freebsd
NAME                              READY     STATUS    RESTARTS   AGE
freebsd-sidecar-556972417-j7zmn   1/1       Running   0          1m
```
4. Attach to the FreeBSD image

```bash
kubectl attach -it freebsd-sidecar-556972417-j7zmn 
```
