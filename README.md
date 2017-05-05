# k8s-ovpn
OpenVPN on a Kubernetes cluster. Save on compute resources by kuberizing services!

# Usage

* Initialise the configuration files and certificates

```bash
# docker
$ mkdir openvpn0 && cd openvpn0
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_genconfig \
    -u udp://VPN.SERVERNAME.COM -C 'AES-256-GCM' -a 'SHA384' -T 'TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384'
$ docker run -e EASYRSA_KEY_SIZE=4096 --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_initpki
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_copy_server_files
```

* Generate client certificate and retrieve client configuration with embedded certificates

```bash
$ docker run -e EASYRSA_KEY_SIZE=4096 --net=none --rm -it -v $PWD:/etc/openvpn kylemanna/openvpn easyrsa build-client-full CLIENTNAME
$ docker run --net=none --rm -v $PWD:/etc/openvpn kylemanna/openvpn ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn
```

## TODO
- [ ] Translate Docker Compose commands from [kylemanna/docker-openvpn/docs/docker-compose.md](https://github.com/kylemanna/docker-openvpn/blob/master/docs/docker-compose.md) into [`kubectl run`](https://kubernetes.io/docs/user-guide/kubectl/v1.6/#run)
