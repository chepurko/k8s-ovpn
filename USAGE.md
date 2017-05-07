# Usage

## Deploying Additional OpenVPN Servers

* Let's create a second OpenVPN server Deployment with a separate configuration. This entails adding some bits to your YAML configurations.

`$ vim ovpn-Services.yaml`

```yaml
...
# run 'shuf -i 49152-65535 -n 1' to find a high, random port
- port: 57156
    targetPort: 57156
    protocol: UDP
    name: openvpn1
...
```

  * Create a new `ovpn1-Deployment.yaml` configuration.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ovpn1
  namespace: ovpn
  labels:
    app: ovpn
    track: stable
spec:
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: openvpn
        app: ovpn
    spec:
      containers:
      - image: chepurko/docker-openvpn
        name: ovpn1
        ports:
        - containerPort: 57156
          name: openvpn1
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        volumeMounts:
        - name: ovpn1-key
          mountPath: /etc/openvpn/pki/private
        - name: ovpn1-cert
          mountPath: /etc/openvpn/pki/issued
        - name: ovpn1-pki
          mountPath: /etc/openvpn/pki
        - name: ovpn1-conf
          mountPath: /etc/openvpn
        - name: ccd1
          mountPath: /etc/openvpn/ccd
      volumes:
      - name: ovpn1-key
        secret:
          secretName: ovpn1-key
          defaultMode: 0600
      - name: ovpn1-cert
        secret:
          secretName: ovpn1-cert
      - name: ovpn1-pki
        secret:
          secretName: ovpn1-pki
          defaultMode: 0600
      - name: ovpn1-conf
        configMap:
          name: ovpn1-conf
      - name: ccd1
        configMap:
          name: ccd1
```

* Initiate a new OpenVPN configuration with the new port number.

```bash
$ mkdir ovpn1 && cd ovpn1
# Modify the crypto algos to your liking and see documentation here
# https://github.com/kylemanna/docker-openvpn/blob/master/docs/paranoid.md
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_genconfig \
    -u udp://VPN.SERVERNAME.COM:57156 \
    -C 'AES-256-GCM' -a 'SHA384' -T 'TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384' \
    -b -n 185.121.177.177 -n 185.121.177.53 -n 87.98.175.85
$ docker run -e EASYRSA_ALGO=ec -e EASYRSA_CURVE=secp384r1 \
    --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_initpki
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_copy_server_files
```

* Generate client ECC certificate and retrieve client configuration with embedded certificates

```bash
$ export CLIENTNAME="your_client_name"
$ docker run -e EASYRSA_ALGO=ec -e EASYRSA_CURVE=secp384r1 \
    --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn easyrsa build-client-full $CLIENTNAME
$ docker run --net=none --rm -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

* Or generate client RSA certificates if your client doesn't support ECC

```bash
$ export CLIENTNAME="your_client_name"
$ docker run --net=none --rm -it -v $PWD:/etc/openvpn chepurko/docker-openvpn easyrsa build-client-full $CLIENTNAME
$ docker run --net=none --rm -v $PWD:/etc/openvpn chepurko/docker-openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

* Fix some file permissions.

```bash
$ sudo chown -R $USER:$USER server/*
```

* Create ConfigMaps and Secrets.

```bash
$ kubectl create secret generic ovpn1-key --from-file=server/pki/private/VPN.SERVERNAME.COM.key
$ kubectl create secret generic ovpn1-cert --from-file=server/pki/issued/VPN.SERVERNAME.COM.crt
$ kubectl create secret generic ovpn1-pki \
    --from-file=server/pki/ca.crt --from-file=server/pki/dh.pem --from-file=server/pki/ta.key
$ kubectl create configmap ovpn1-conf --from-file=server/
$ kubectl create configmap ccd1 --from-file=server/ccd

```

* Bring up the OpenVPN server in your Kubernetes cluster.

```bash
$ kubectl apply -f ../ovpn-Service.yaml
$ kubectl apply -f ../ovpn1-Deployment.yaml
```

* Now the `Service` object you created, which is acting as a load balancer, will direct traffic between two ports (1194 and 57156, in this case) and their respective `Deployments`, on the same address (VPN.SERVERNAME.COM).
