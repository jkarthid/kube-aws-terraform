

## Kubernetes PKI vault backend

### Kubernetes Certificate Management

There are two certificate authorities used to sign certificates for secure communication between Kubernetes components:

- etcd-member: for etcd and kubernetes master (api-server, scheduler, and kubectl). 
- kube-apiserver: for kubernetes master and nodes (kubelet, kube-proxy).

Vault server will be the first one to build. To verify Vault server's readiness, e.g. if CLUSTER_NAME environment is _kube-cluster_, you should see _etcd_ and _kube_ PKI backend. 

```
$ ssh-add ~/.ssh/<cluster-name>-vault.pem
$ cd resources/vault
$ make ssh
Last login: Thu Jan 26 19:02:40 UTC 2017 from xx.xx.xx.xx on pts/0
Container Linux by CoreOS beta (1248.4.0)
core@ip-10-240-6-26-kube-cluster-vault ~ $ sudo su -
root@ip-10-240-6-26-kube-cluster-vault ~ $ vault mounts
Path                            Type       Default TTL  Max TTL    Description
cubbyhole/                      cubbyhole  n/a          n/a        per-token private secret storage
kube-cluster/pki/etcd-member/     pki        system       315360000  kube-cluster/pki/etcd-member Root CA
kube-cluster/pki/kube-apiserver/  pki        system       315360000  kube-cluster/pki/kube-apiserver Root CA
secret/                         generic    system       system     generic secret storage
sys/                            system     n/a          n/a        system endpoints used for control, policy and debugging

```
As you can see, there are two PKI mounts for kubernetes cluster.

* Vault audit log

Vault runs as a container, and the audit log is enabled:
```
$ vault audit-list
Path   Type  Description  Options
file/  file               path=/vault/logs/vault_audit.log
```
The log is mounted on host as /var/log/vault/vault_audit.log.

* Kubernetes certificates

On etcd servers, masters, and nodes, certificates are dynamically generated by __install_cert__ system unit on every reboot.

etcd certificates:
```
$ cd resources/etcd
$ make ssh
core@ip-10-240-1-22-etcd ~ $ ls -1 /etc/etcd/certs
etcd-server-ca.pem
etcd-server-key.pem
etcd-server.pem
kube-bundle.certs
```

Controller certificates:

```
core@ip-10-240-10-136-master /etc/etcd/certs $ ls -1
etcd-server-ca.pem
etcd-server-key.pem
etcd-server.pem
```
```
ls -1 /var/lib/kubernetes/
admin-key.pem
admin.pem
kube-apiserver-ca.pem
kube-apiserver-key.pem
kube-apiserver.pem
service-account-key.pem
```

Node certificates:

```
core@ip-10-240-5-79-node ~ $ ls -1 /var/lib/kubelet/
kube-apiserver-ca.pem
kubeconfig
kubelet-key.pem
kubelet.pem
```
* Vault PKI server

This implementation has an internal AWS ELB for vault service. The Route53 DNS name is https://vault.cluster.internal.
Vault server requires TLS connections from clients. Vault server self-signed CA is managed by Terraform PKI module. The certificate authority is used to sign Vault server cert and validate vault client. 

To re-generate CA:

- Run make pki
- On vault server `systemctl stop vault; systemctl start vault; /opt/etc/vault/scripts/init-unseal.sh`
- On each vault client servers (etcd, master, node), `systemctl start s3sync`, which will download new CA cert file.

To view certificate information:

```
$ cd resources/pki
$ make output



