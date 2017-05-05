# k8s-ovpn
OpenVPN on a Kubernetes cluster. Save on compute resources by kuberizing services!

# Usage

* Initialise the configuration files and ECC certificates
  * **Run these commands on your workstation.** You are creating a directory with OpenVPN configuration and sensitive PKI files. [Docker](https://docs.docker.com/engine/installation/) is required.

```bash
# docker
$ mkdir openvpn0 && cd openvpn0
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_genconfig \
    -u udp://VPN.SERVERNAME.COM -C 'AES-256-GCM' -a 'SHA384' -T 'TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384'
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

* Bring up the OpenVPN server in your Kubernetes cluster.

```bash

```

## TODO
- [ ] Translate Docker Compose commands from [kylemanna/docker-openvpn/docs/docker-compose.md](https://github.com/kylemanna/docker-openvpn/blob/master/docs/docker-compose.md) into [`kubectl run`](https://kubernetes.io/docs/user-guide/kubectl/v1.6/#run)
