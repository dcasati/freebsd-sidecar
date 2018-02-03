# freebsd-sidecar
A very experimental way to run FreeBSD as a Kubernetes workload 

# WARNING

This is a _very_ experimental work, not suitable for production yet. The initial drive for this experience was curiosity: can that be done ? Can I run FreeBSD as a workload inside of a Kubernetes cluster?

## TODO
 - Networking
 - Make a smaller image with crunchgen

# Solution

The solution involves running FreeBSD as a qemu workload with Arch Linux being the base OS.

# Prerequisites

- FreeBSD qcow2 image: 
  ```bash
  wget http://ftp.freebsd.org/pub/FreeBSD/releases/VM-IMAGES/11.0-RELEASE/amd64/Latest/FreeBSD-11.0-RELEASE-amd64.qcow2.xz
  ```
- unpack the image

  ```bash
  xz -d FreeBSD-11.0-RELEASE-amd64.qcow2.xz
  ```

## The Dockerfile

```bash
cat << EOF > Dockerfile
FROM alpine
RUN apk update && apk add qemu-system-x86_64 libvirt qemu-img bash

COPY FreeBSD-11.0-RELEASE-amd64.qcow2 /base_image/FreeBSD-11.0-RELEASE-amd64.qcow2
CMD ["/usr/bin/qemu-system-x86_64", "-curses", "/base_image/FreeBSD-11.0-RELEASE-amd64.qcow2"]
EOF
```

## Create the Docker Image

```bash
 docker build -t dcasati/inceptionbsd  .
 ```
> Output
```bash
dcasati@ubuntu:~/freebsd-sidecar$ docker build -t dcasati/inceptionbsd  .
Sending build context to Docker daemon 1.862 GB
Step 1/4 : FROM alpine
 ---> e21c333399e0
Step 2/4 : RUN apk update && apk add qemu-system-x86_64 libvirt qemu-img bash
 ---> Using cache
 ---> d7a082846326
Step 3/4 : COPY FreeBSD-11.0-RELEASE-amd64.qcow2 /base_image/FreeBSD-11.0-RELEASE-amd64.qcow2
 ---> daf9217211a3
Removing intermediate container 83aeaf776a1c
Step 4/4 : CMD /usr/bin/qemu-system-x86_64 -curses /base_image/FreeBSD-11.0-RELEASE-amd64.qcow2
 ---> Running in 2bf4280c5111
 ---> e48a48352469
Removing intermediate container 2bf4280c5111
Successfully built e48a48352469
```

## Testing the new image locally

```bash
docker run -it  dcasati/inceptionbsd
```

## Deploying on Kubernetes

1. Create a deployment file

```bash
cat << EOF > deployment.yml
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
