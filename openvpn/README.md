# --- This is modified openvpn Helm repository for me ---

This repository is made for my work.
In this hands-on, openvpn server has an multus ip by multus CNI.
Server is connected another network with L3 switch, so Server must have routing tables for other networks that L3 switch know by Multus interface.
> route add -net [network cidr] dev [multus i/f]

## Preq - Kor
## 사용방법
#### 1. Values.service.nodePort
이미 배포되어 있는 노드포트가 곂치지 않게 변경.
#### 2. Values.route : Multus CNI쓸경우에만 적용
#### 2-1. Values.podAnnotations
Multus CNI쓸 경우 해당 NAD를 그 namespace에 배포가  미리되어 있어야 하며, 이름은 "openvpn-datanet"으로 배포하기.
* 이 이름은 values파일의 podAnnotaion에 정의되어 있고 변경가능.
#### 2-2. values.route.interface
Multus i/f를 통해 라우팅이 필요할 경우 template폴더의 deployment.yaml내부에서 pod command에 추가하기 
values.route.interface의 이름도 podAnnotaion에 설정된 interface 이름과 맞춰주기
* 네트워크 개수에 따라 직접 추가가 필요함. templates/openvpn-deployment.yaml command field를 참조하면됨.
#### 3.  Values.service.lab_public_ip
해당 pod의 /etc/openvpn/newClient.sh를 실행할 때 자동으로 생성되는 client ovpn파일이 만들어질 때 사용되며, client에서 연결하고 싶은 openvpnserver의 ip를 설정하는 것  으로, host의 ip이거나 public ip임
* ovpn 파일은 자동으로 생성되더라도 한번씩 체크할 필요가 있음
#### 4. Values.imagePullSecretName
사용되는 이미지가 private registry에 있으면 인증을 위해 imagepullsecret을 사용하면됨. 해당 네임스페이스에 미리 배포되어 있어야함
* 이 파일을 참고하면됨 "csp-docker-registry-secret.txt"
#

## Preq - Eng
#### 1. change Nodeport. 노드포트가 곂치지 않게 변경.
#### 2. For Multus CNI, Multus CNI쓸경우에만 적용
#### 2-1. if you make NetworkAttachmentDefinition CRD for Multus CNI, 
make NAD first as "openvpn-datanet in deploying namespace.
And, set IP on podAnnotaion field.
#### 2-2.  Add routing in .Values.route,
And then match interface name with podAnnotition's interface.
* If you use routing better, check the file templates/openvpn-deployment.yaml command field.
#### 3.  Set "lab_public_ip" in values.file that is same as your host server IP or Real Public IP.
This is used when client ovpn file will be made.
#### 4.  if you deploy with private registry that need secrets for auth, Use ImagePullSecrets in values.file.
But Before you use, Make ImagePullSecrets Object first in deploying namespace and then write the name in your values.file.
  * refer to the file "ncp-docker-registry-secret.txt".

##
# --- End ---
## Below is original READ.me
# ⚠️ Repo Archive Notice

As of Nov 13, 2020, charts in this repo will no longer be updated.
For more information, see the Helm Charts [Deprecation and Archive Notice](https://github.com/helm/charts#%EF%B8%8F-deprecation-and-archive-notice), and [Update](https://helm.sh/blog/charts-repo-deprecation/).

# Helm chart for OpenVPN
This chart will install an [OpenVPN](https://openvpn.net/) server inside a kubernetes cluster.  New certificates are generated on install, and a script is provided to generate client keys as needed.  The chart will automatically configure dns to use kube-dns and route all network traffic to kubernetes pods and services through the vpn.  By connecting to this vpn a host is effectively inside a cluster's network.

## DEPRECATION NOTICE

This chart is deprecated and no longer supported.

### Uses
The primary purpose of this chart was to make it easy to access kubernetes services during development.  It could also be used for any service that only needs to be accessed through a vpn or as a standard vpn.

## Usage

```bash
helm repo add stable http://storage.googleapis.com/kubernetes-charts
helm install stable/openvpn
```

Wait for the external load balancer IP to become available.  Check service status via: `kubectl get svc`

Please be aware that certificate generation is variable and may take some time (minutes).
Check pod status, replacing `$HELM_RELEASE` with the name of your release, via:

```bash
POD_NAME=$(kubectl get pods -l "app=openvpn,release=$HELM_RELEASE" -o jsonpath='{.items[0].metadata.name}') \
&& kubectl logs "$POD_NAME" --follow
```

When all components of the openvpn chart have started use the following script to generate a client key:


```bash
#!/bin/bash

if [ $# -ne 3 ]
then
  echo "Usage: $0 <CLIENT_KEY_NAME> <NAMESPACE> <HELM_RELEASE>"
  exit
fi

KEY_NAME=$1
NAMESPACE=$2
HELM_RELEASE=$3
POD_NAME=$(kubectl get pods -n "$NAMESPACE" -l "app=openvpn,release=$HELM_RELEASE" -o jsonpath='{.items[0].metadata.name}')
SERVICE_NAME=$(kubectl get svc -n "$NAMESPACE" -l "app=openvpn,release=$HELM_RELEASE" -o jsonpath='{.items[0].metadata.name}')
SERVICE_IP=$(kubectl get svc -n "$NAMESPACE" "$SERVICE_NAME" -o go-template='{{range $k, $v := (index .status.loadBalancer.ingress 0)}}{{$v}}{{end}}')
kubectl -n "$NAMESPACE" exec -it "$POD_NAME" /etc/openvpn/setup/newClientCert.sh "$KEY_NAME" "$SERVICE_IP"
kubectl -n "$NAMESPACE" exec -it "$POD_NAME" cat "/etc/openvpn/certs/pki/$KEY_NAME.ovpn" > "$KEY_NAME.ovpn"
```

In order to revoke certificates in later steps:
```bash
#!/bin/bash

if [ $# -ne 3 ]
then
  echo "Usage: $0 <CLIENT_KEY_NAME> <NAMESPACE> <HELM_RELEASE>"
  exit
fi

KEY_NAME=$1
NAMESPACE=$2
HELM_RELEASE=$3
POD_NAME=$(kubectl get pods -n "$NAMESPACE" -l "app=openvpn,release=$HELM_RELEASE" -o jsonpath='{.items[0].metadata.name}')
kubectl -n "$NAMESPACE" exec -it "$POD_NAME" /etc/openvpn/setup/revokeClientCert.sh $KEY_NAME
```

The entire list of helper scripts can be found on [templates/config-openvpn.yaml](templates/config-openvpn.yaml)

Be sure to change `KEY_NAME` if generating additional keys.  Import the .ovpn file into your favorite openvpn tool like tunnelblick and verify connectivity.

## Configuration
The following table lists the configurable parameters of the `openvpn` chart and their default values,
and can be overwritten via the helm `--set` flag.

Parameter | Description | Default
---                                  | ---                                                                  | ---
`replicaCount`                       | amount of parallel openvpn replicas to be started                    | `1`
`updateStrategy`                     | update strategy for deployment                                       | `{}`
`image.repository`                   | `openvpn` image repository                                           | `jfelten/openvpn-docker`
`image.tag`                          | `openvpn` image tag                                                  | `1.1.0`
`image.pullPolicy`                   | Image pull policy                                                    | `IfNotPresent`
`imagePullSecretName`                | Docker registry pull secret name                                     |
`service.type`                       | k8s service type exposing ports, e.g. `NodePort`                     | `LoadBalancer`
`service.externalPort`               | TCP port reported when creating configuration files                  | `443`
`service.internalPort`               | TCP port on which the service works                                  | `443`
`service.hostPort`                   | Expose openvpn directly using host port                              | `nil`
`service.nodePort`                   | NodePort value if service.type is `NodePort`                         | `nil` (auto-assigned)
`service.clusterIP`                  | clusterIP value if service.type is `ClusterIP`                       | `nil`
`service.externalIPs`                | External IPs to listen on                                            | `[]`
`resources.requests.cpu`             | OpenVPN cpu request                                                  | `300m`
`resources.requests.memory`          | OpenVPN memory request                                               | `128Mi`
`resources.limits.cpu`               | OpenVPN cpu limit                                                    | `300m`
`resources.limits.memory`            | OpenVPN memory limit                                                 | `128Mi`
`readinessProbe.initialDelaySeconds` | Time to wait to start first probe                                    | `5`
`readinessProbe.periodSeconds`       | Interval of readiness probe                                          | `5`
`readinessProbe.successThreshold`    | Minimum consecutive successes for probe to be considered healthy     | `2`
`persistence.enabled`                | Use a PVC to persist configuration                                   | `true`
`persistence.subPath`                | Subdirectory of the volume to mount at                               | `nil`
`persistence.existingClaim`          | Provide an existing PersistentVolumeClaim                            | `nil`
`persistence.storageClass`           | Storage class of backing PVC                                         | `nil`
`persistence.accessMode`             | Use volume as ReadOnly or ReadWrite                                  | `ReadWriteOnce`
`persistence.size`                   | Size of data volume                                                  | `2M`
`podAnnotations`                     | Key-value pairs to add as pod annotations                            | `{}`
`openvpn.OVPN_NETWORK`               | Network allocated for openvpn clients                                | `10.240.0.0`
`openvpn.OVPN_SUBNET`                | Network subnet allocated for openvpn                                 | `255.255.0.0`
`openvpn.OVPN_PROTO`                 | Protocol used by openvpn tcp or udp                                  | `tcp`
`openvpn.OVPN_K8S_POD_NETWORK`       | Kubernetes pod network (optional)                                    | `10.0.0.0`
`openvpn.OVPN_K8S_POD_SUBNET`        | Kubernetes pod network subnet (optional)                             | `255.0.0.0`
`openvpn.OVPN_K8S_SVC_NETWORK`       | Kubernetes service network (optional)                                | `nil`
`openvpn.OVPN_K8S_SVC_SUBNET`        | Kubernetes service network subnet (optional)                         | `nil`
`openvpn.DEFAULT_ROUTE_ENABLED`      | Push a route which openvpn sets by default                           | `true`
`openvpn.dhcpOptionDomain`           | Push a `dhcp-option DOMAIN` config                                   | `true`
`openvpn.serverConf`                 | Lines appended to the end of the server configuration file (optional)| `nil`
`openvpn.clientConf`                 | Lines appended into the client configuration file (optional)         | `nil`
`openvpn.redirectGateway`            | Redirect all client traffic through VPN                              | `true`
`openvpn.useCrl`                     | Use/generate a certificate revocation list (crl.pem)                 | `false`
`openvpn.taKey`                      | Use/generate a ta.key file for hardening security                    | `false`
`openvpn.cipher`                     | Override the default cipher                                          | `nil` (OpenVPN default)
`openvpn.istio.enabled`              | Enables istio support for openvpn clients                            | `false`
`openvpn.istio.proxy.port`           | Istio proxy port                                                     | `15001`
`openvpn.iptablesExtra`              | Custom iptables rules for clients                                    | `[]`
`openvpn.ccd.enabled`                | Enable creation and mounting of CCD config                           | `false`
`openvpn.ccd.config`                 | CCD configuration (see below)                                        | `{}`
`nodeSelector`                       | Node labels for pod assignment                                       | `{}`
`tolerations`                        | Tolerations for node taints                                          | `[]`
`ipForwardInitContainer`             | Add privileged init container to enable IPv4 forwarding              | `false`

This chart has been engineered to use kube-dns and route all network traffic to kubernetes pods and services,
to disable this behaviour set `openvpn.OVPN_K8S_POD_NETWORK` and `openvpn.OVPN_K8S_POD_SUBNET` to `null`.

If openvpn.OVPN_K8S_SVC_NETWORK and openvpn.OVPN_K8S_SVC_SUBNET are defined, an extra route for services subnet will be added.

#### Note: As configured the chart will create a route for a large 10.0.0.0/8 network that may cause issues if that is your local network.  If so tweak this value to something more restrictive.  This route is added, because GKE generates pods with IPs in this range.

### Certificates

New certificates are generated with each deployment, if *keystoreSecret* is not defined.
If persistence is enabled certificate data will be persisted across pod restarts.
Otherwise new client certs will be needed after each deployment or pod restart.

Certificates can be passed in secret, which name is specified in *openvpn.keystoreSecret* value.
Create secret as follows:

```bash
kubectl create secret generic openvpn-keystore-secret --from-file=./server.key --from-file=./ca.crt --from-file=./server.crt --from-file=./dh.pem [--from-file=./crl.pem] [--from-file=./ta.key]
```

You can deploy temporary openvpn chart, create secret from generated certificates, and then re-deploy openvpn, providing the secret.
Certificates can be found in openvpn pod in the following files:

 `/etc/openvpn/certs/pki/private/server.key`
 `/etc/openvpn/certs/pki/ca.crt`
 `/etc/openvpn/certs/pki/issued/server.crt`
 `/etc/openvpn/certs/pki/dh.pem`

If openvpn.useCrl is set:

 `/etc/openvpn/certs/pki/crl.pem`

And optionally (see openvpn.taKey setting):

 `/etc/openvpn/certs/pki/ta.key`

Note: using mounted secret makes creation of new client certificates impossible inside openvpn pod, since easyrsa needs to write in certs directory, which is read-only.

### Client specific rules and access policies

You can enable CCD using `openvpn.ccd.enabled` and set the config in `openvpn.ccd.config` to use [OpenVPN client specific rules and access policies](https://openvpn.net/community-resources/configuring-client-specific-rules-and-access-policies/)

For example, if you want to give fixed IP addresses to clients 'johndoe' and 'janedoe':

```
openvpn:
  ccd:
    enabled: true
    config:
      johndoe: "ifconfig-push 10.240.100.10 10.240.100.11"
      janedoe: "ifconfig-push 10.240.100.20 10.240.100.21"
```

For more options see the OpenVPN documentation. Note that the IPs provided here depend on the type of topology you use.


## Issues

### 1. Routing / ip_forward

Issue: https://github.com/helm/charts/issues/6398

If routes look correct on the client but data is not returning from the vpn then the kubernetes node running openvpn may not have ip_forward enabled.  Set the `ipForwardInitContainer` value to `true` to run an init container that enables ip forwarding.

### 2. Ubuntu/systemd-resolved DNS

Recent Ubuntu releases use systemd-resolved for DNS which by default [won't honor/apply DNS settings from openvpn](https://askubuntu.com/questions/1032476/ubuntu-18-04-no-dns-resolution-when-connected-to-openvpn).

Install the openvpn-systemd-resolved package (`apt install openvpn-systemd-resolved`) and add the following settings to the client ovpn file.

```
script-security 2
up /etc/openvpn/update-systemd-resolved
up-restart
down /etc/openvpn/update-systemd-resolved
down-pre
dhcp-option DOMAIN-ROUTE .
```

If all of your clients are Ubuntu you can set the `openvpn.clientConf` value when deploying this chart to have these lines added to all generated client ovpn files:

```yaml
openvpn:
  clientConf: |
    script-security 2
    up /etc/openvpn/update-systemd-resolved
    up-restart
    down /etc/openvpn/update-systemd-resolved
    down-pre
```

### 3. Ubuntu Networking GUIs

Importing the client ovpn file from either of the Ubuntu network/connection management GUIs (Settings or Advanced Networking app) do not successfully import all settings.  They seem to remove important parts of the configuration (DNS and Domains).  The most reliable method of initiating the connection is to run `sudo openvpn --config <FILE>`.
