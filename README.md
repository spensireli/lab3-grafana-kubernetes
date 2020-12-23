# Lab3 A Persistent Grafana Server with TLS Encryption
This project will walk through how to deploy a Grafana server that is persistent with TLS encryption.

*Note this lab is not for production use.*


### Lab configuration
Below you will find the specifications for the environment used to run this lab. I am confident the lab is able to run on much less hardware or even on a set of Raspberry Pi's.

#### Virtualization Environment
| Physical Server  | Hypervisor | Physical CPU | Physical Memory | Physical Storage |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| Dell R420  | ProxMox Virtual Environment 6.2-4 | 8  | 32G | 2TB RAID 5 |
| Dell R420  | ProxMox Virtual Environment 6.2-4 | 24  | 32G | 4TB RAID 5 |

#### Host Machines

| Hostname  | Operating System | vCPU | Memory | Storage |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| kubernetes-controller-1  | Ubuntu 20.04  | 4  | 8G | 100G |
| kubernetes-worker-1  | Ubuntu 20.04  | 2  | 4G  | 32G |
| kubernetes-worker-2  | Ubuntu 20.04  | 2  | 4G  | 32G |
| kubernetes-worker-3  | Ubuntu 20.04  | 2  | 4G  | 32G |
| nfs-server-1  | CentOS 7 | 4  | 4G  | 250G |

### Pre-requisites
- A Kubernetes cluster running v1.19.4.
  - This lab has 3 worker nodes with 4GiB memory and 2 vCPU's.
- Access to the Kubernetes cluster.
- Kubectl already configured for use.
- An already configured NFS server ready to use for persistent storage.

## Installation

It is suggested that you go through the first lab before executing this one. This lab requires that you have an NFS server to mount your persistent volume to.  

#### TLS Key Creation

Before we begin the lab lets create a selfsigned TLS certificate for our Grafana server. This will allow encryption in transit from the client to the server.

```bash
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout privateKey.key -out certificate.crt
 ```

Upon completion of the command you will be left with two files, `privateKey.key` and `certificate.crt`. We will reference these later on in the Grafana server creation.

#### Grafana Namespace

First we must create a namespace for our Grafana server to live. Create a file called `grafana-ns.yaml`.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: grafana
```

Then create the namespace.

```bash
$ kubectl create -f grafana-ns.yaml
namespace/grafana created

You can confirm it was created by listing the namespaces.

```bash
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   13d
grafana           Active   28s
kube-node-lease   Active   13d
kube-public       Active   13d
kube-system       Active   13d
```

#### Grafana Secrets

Before we begin lets create two secrets that we will use later on in this configuration. Create a file called `grafana-secrets.yaml`. In this file paste the following.

```yaml
---
apiVersion: v1
metadata:
  name: grafana-tls
  namespace: grafana
kind: Secret
type: kubernetes.io/tls
data:
  tls.crt: < YOUR BASE64 ENCODED CERT >
  tls.key: < YOUR BASE64 ENCODED KEY >
```

As you may have noted the we must add the base64 encoded values of our certificate and key.

To get the base64 encoded values for both of these perform the following against the files we created earlier.

```bash
$ cat privateKey.key |base64
$ cat certificate.crt |base64
```

Copy the output of the `.crt` and paste it into `< YOUR BASE64 ENCODED CERT >`.
Copy the output of the `.key` and paste it into `< YOUR BASE64 ENCODED KEY >`.

Now create the secrets by running the following.
```bash
$kubectl create -f grafana-secrets.yaml
```


#### Persistent Volume

Next we will add a persistent volume for our Grafana server to use. This will allow us to keep the data even if the pod is destroyed.

Like in the [NFS Lab 1](https://github.com/spensireli/lab1-kubernetes-nfs) we will create a new directory on the nfsshare called `pv0006`. This is where all the Grafana data will be stored.

Create a file called `grafana-pv.yaml` and add the following, changing the NFS server address to your NFS server.

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0006-grafana
  namespace: grafana
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: pv0006-grafana
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfsshare/pv0006/grafana
    server: 192.168.1.195
```


Create the persistent volume.

```bash
$ kubectl create -f grafana-pv.yaml
persistentvolume/pv0006-grafana created
```

You can confirm that the persistent volume exists by running the following command.

```bash
$ kubectl  get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS     REASON   AGE
pv0006-grafana   20Gi       RWO            Recycle          Available           pv0006-grafana            35s
```



#### Persistent Volume Claim

Now lets create the persistent volume claim for the Grafana volume that we have just created. Create a file called `grafana-pvc.yaml` and add the following.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv0006-grafana
  namespace: grafana
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: pv0006-grafana
  resources:
    requests:
      storage: 20Gi
```

Create the persistent volume claim.

```bash
$ kubectl create -f grafana-pvc.yaml
persistentvolumeclaim/pv0006-grafana created
```

You can validate that the persistent volume claim was created by running the following command.
```bash
$ kubectl get pvc -n grafana
NAME             STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS     AGE
pv0006-grafana  Bound    pv0006-grafana  20Gi       RWO            pv0006-grafana   61s
```

We can see that our persistent volume claim is bound.


#### Grafana Configmap

For our grafana server there is a configuration file called `grafana.ini`. We have made a few changes to this file and would like to be able to call it within our deployment. The configuration file is not necessarily sensitive information, and is not worth of mounting an entire data directory to use it. For that reason we will use a `configmap`.

One of the easiest ways to create a configmap is using the `--from-file` command. Try this against the `grafana.ini` by running the command below.

```bash
$ kubectl -n grafana create configmap grafana-config --from-file=./grafana.ini
```

You will note that it created the configmap from the configuration file we have specified. To see the `YAML` you can simply run the following command.

```bash
$ kubectl -n grafana get configmap grafana-config -o yaml
```

This will output a YAML file that you can save and apply in the future.

#### Grafana Deployment

Once we have the persistent volume and claim created we can now deploy the container.

What we are doing is configuring a container with a single replica with a label `grafana`.

This is going to be deployed to the `grafana` namespace that we first created. We are running the container on port `3000`. The important component here for persistent data between deletion is the `volumeMounts`.

You may note that we have defined our volume mount as `grafana-pv-storage` which it gets from the `volumes` section. The `volumes` section declares this as the persistent volume claim we created earlier called `pv0006-grafana`.

This volume is mounted inside the container to `/var/lib/grafana`.

There is a secret with two entries that we are calling. This secret the `grafana-tls` secret we created earlier, first we are calling the `tls.crt` entry and setting the path to `certificate.csr` and secondly we call the `tls.key` entry and set it to `private_key.key` in the container. The certificate is mounted to the directory `/etc/ssl/grafana/crt/` and the key is mounted to `/etc/ssl/grafana/key/` within the container.

Lastl

Create a file called `grafana-deployment.yaml` and add the following.


```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
        namespace: grafana
    spec:
      containers:
      - name: grafana
        env:
        - name: GF_INSTALL_PLUGINS
          value: "grafana-clock-panel,grafana-simple-json-datasource"
        - name: GF_PATHS_DATA
          value: "/var/lib/grafana"
        image: grafana/grafana:7.3.6-ubuntu
        ports:
          - name: https-port
            containerPort: 3000
        volumeMounts:
          - name: grafana-pv-storage
            mountPath: /var/lib/grafana
          - name: grafana-config
            mountPath: /etc/grafana/
          - name: grafana-cert
            mountPath: /etc/ssl/grafana/crt/
          - name: grafana-key
            mountPath: /etc/ssl/grafana/key/
      volumes:
        - name: grafana-pv-storage
          persistentVolumeClaim:
            claimName: pv0006-grafana
        - name: grafana-config
          configMap:
            name: grafana-config
        - name: grafana-cert
          secret:
            secretName: grafana-tls
            items:
              - key: tls.crt
                path: certificate.csr
        - name: grafana-key
          secret:
            secretName: grafana-tls
            items:
              - key: tls.key
                path: private_key.key
```

Deploy the grafana container.

```bash
$ kubectl create -f grafana-deployment.yaml
deployment.apps/grafana created
```

You can confirm that it has been created by running the following commands.
```bash
$ kubectl -n grafana get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE                  NOMINATED NODE   READINESS GATES
grafana-6ff8d69b-7rl9d   1/1     Running   0          56s   10.244.1.9   kubernetes-worker-1   <none>           <none>
```


#### grafana Service

Here we are creating a service to expose the container port that we declared in the deployment. We declare the port that the service is running on in the container `3000` and what we would like it to be broadcast as on the hosts `nodePort` which is port `31443`.


Create a file called `grafana-svc.yaml` and add the following.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: grafana
spec:
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 31443
  selector:
    app: grafana
```

Create the service by running the following commands.

```bash
$ kubectl create -f grafana-svc.yaml
service/grafana created
```

You can confirm the services exist by running the following commands.

```bash
$ kubectl -n grafana get services
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
grafana   NodePort   10.97.137.186   <none>        3000:31443/TCP   31m

```
#### Accessing The Grafana Server
You can now access the Grafana server by connecting to port `31443` of the IP address of the worker node that the container lives.

To get the node that the container lives on run the following command.

```bash
$ kubectl -n grafana get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP           NODE                  NOMINATED NODE   READINESS GATES
grafana-6ff8d69b-7rl9d   1/1     Running   0          5m14s   10.244.1.9   kubernetes-worker-1   <none>           <none>
```

We can see that the node name is `kubernetes-worker-1`.

Now run the following command to get details about the node.

```bash
$ kubectl get node kubernetes-worker-1 -o wide
NAME                  STATUS   ROLES    AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION     CONTAINER-RUNTIME
kubernetes-worker-1   Ready    <none>   13d   v1.19.3   192.168.1.200   <none>        Ubuntu 20.04 LTS   5.4.0-53-generic   docker://19.3.8
```

Open your web browser and navigate to the address and port `31443`. For example `https://192.168.1.200:31443`.

You will be prompted to log in for the first time which can be done by using the default credentials `admin` and `admin`. You will be prompted to change your password.


### To Do

- Add images for Grafana configuration
- Add uninstall guide
