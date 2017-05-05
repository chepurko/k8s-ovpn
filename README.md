# k8s-ovpn
OpenVPN on a Kubernetes cluster. Save on compute resources by kuberizing services!

# Usage

* Initialise the configuration files and ECC certificates
  * **Run these commands on your workstation.** You are creating a directory with OpenVPN configuration and sensitive PKI files. [Docker](https://docs.docker.com/engine/installation/) is required.

```bash
# docker
$ mkdir openvpn0 && cd openvpn0
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_genconfig \
    -u udp://VPN.SERVERNAME.COM \
    -C 'AES-256-GCM' -a 'SHA384' -T 'TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384' \
    -b -n 185.121.177.177 -n 185.121.177.53 -n 87.98.175.85
$ docker run -e EASYRSA_ALGO=ec -e EASYRSA_CURVE=secp384r1 \
    --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_initpki
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_copy_server_files
```

* Generate client ECC certificate and retrieve client configuration with embedded certificates

```bash
$ export CLIENTNAME="your_client_name"
$ docker run -e EASYRSA_ALGO=ec -e EASYRSA_CURVE=secp384r1 \
    --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn easyrsa build-client-full $CLIENTNAME
$ docker run --net=none --rm -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

* Or generate client RSA certificates if your client doesn't support ECC

```bash
$ export CLIENTNAME="your_client_name"
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn easyrsa build-client-full $CLIENTNAME
$ docker run --net=none --rm -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

* Prepare the namespace and some file permissions.

```bash
$ kubectl apply -f 00-namespace.yaml
$ kubectl config set-context $(kubectl config current-context) --namespace=ovpn
# Validate it
$ kubectl config view | grep namespace:
$ sudo chown -R $USER:$USER server/*
```

* Create ConfigMaps and Secrets.

```bash
$ kubectl create secret generic ovpn0-key --from-file=server/pki/private/VPN.SERVERNAME.COM.key
$ kubectl create secret generic ovpn0-cert --from-file=server/pki/issued/VPN.SERVERNAME.COM.crt
$ kubectl create secret generic ovpn0-pki \
    --from-file=server/pki/ca.crt --from-file=server/pki/dh.pem --from-file=server/pki/ta.key
$ kubectl create configmap openvpn-conf --from-file=server/
$ kubectl create configmap ccd --from-file=server/ccd

```

* Bring up the OpenVPN server in your Kubernetes cluster.

```bash
$ kubectl apply -f ../ovpn-Deployment.yaml
```

## TODO
- [ ] Fix "Options error: Unrecognized option or missing or extra parameter(s) in /etc/openvpn/openvpn.conf:30: push (2.4.1)" - due to missing quotes in openvpn.conf
