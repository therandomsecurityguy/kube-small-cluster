# kubernetes-small-cluster
Kubernetes small cluster build for running on single NIC baremetal systems such as [Intel NUC](www.intel.com/content/www/us/en/nuc/overview.html) or [Gigabyte Brix](http://www.gigabyte.us/brix/2015brix/) 

Based on https://github.com/coreos/coreos-kubernetes

## Why?

This is used for building small clusters based on spare servers, or in this case Intel NUC units, without incurring compute charges from cloud providers.

## Create configuration:

Customize build parameters in `build-data.sh`

```
./build-data.sh
```

## CoreOS installation

You can use the install method of choice, but for simplicity we are using a USB baremetal install:

Mount the USB drive:

```
mount /dev/sdb1 /mnt
```

Download CoreOS, update permissions, and install on USB drive:

```
wget https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install
chmod +x coreos-install
sudo ./coreos-install -d /dev/sda -v 899.1.0 -c /mnt/user-data-<node ip>
```

Reboot system and choose to boot from USB drive.

## Configure kubectl

```
kubectl config set-cluster nuc --server=https://10.10.10.10:443 --certificate-authority=${PWD}/ssl/ca.pem
kubectl config set-credentials nuc-admin --certificate-authority=${PWD}/ssl/ca.pem --client-key=${PWD}/ssl/admin-key.pem --client-certificate=${PWD}/ssl/admin.pem
kubectl config set-context nuc --cluster=nuc --user=nuc-admin
kubectl config use-context nuc
```

```
$ kubectl get nodes
NAME       LABELS                            STATUS    AGE
10.10.10.10   kubernetes.io/hostname=10.10.10.10   Ready     1h
10.10.10.11   kubernetes.io/hostname=10.10.10.11   Ready     1h
10.10.10.12   kubernetes.io/hostname=10.10.10.12   Ready     1h
```

## Client certificate installation

To access the apiserver url (https://10.10.10.10) you'll need a client certificate. Without one you'll see this:
```
$ curl https://10.10.10.10/ -v
*   Trying 10.10.10.10...
* Connected to 10.10.10.10 (10.10.10.10) port 443 (#0)
* WARNING: using IP address, SNI is being disabled by the OS.
* TLS 1.2 connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate: kube-controller
* Server certificate: kube-ca
> GET / HTTP/1.1
> Host: 10.10.10.10
> User-Agent: curl/7.43.0
> Accept: */*
>
< HTTP/1.1 401 Unauthorized
< Content-Type: text/plain; charset=utf-8
< Date: Mon, 11 Jan 2016 18:16:31 GMT
< Content-Length: 13
<
Unauthorized
* Connection #0 to host 10.10.10.10 left intact
```

```
curl https://10.10.10.10/  -E ssl/worker.p12:<your password> --cacert ssl/ca.pem
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/healthz",
    "/healthz/ping",
    "/logs/",
    "/metrics",
    "/resetMetrics",
    "/swagger-ui/",
    "/swaggerapi/",
    "/ui/",
    "/version"
  ]
}
```

To fix this issue you need to install the generated certificate `worker.p12` and `ca.pem` located in the ssl directory.

## Addon installation:
```
kubectl create -f kube-manifests/kube-dns-rc.yaml
kubectl create -f kube-manifests/kube-dns-svc.yaml
kubectl create -f kube-manifests/kube-ui-rc.yaml
kubectl create -f kube-manifests/kube-ui-svc.yaml
```

Now you'll be able to access the Kubernetes UI located in `https://10.10.10.10/api/v1/proxy/namespaces/kube-system/services/kube-ui/#/dashboard/`

