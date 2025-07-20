# Setup guide

1. Install base image
2. Change passwords (`rock` and `root`)
   1. Check that `rock` got ownership of `/home/rock`
3. Enable Wi-Fi
4. Connect with `ssh` from PC
5. Enable Bluetooth (necessary?)
6. Install `less`
7. Install `git`
8. Install `snapd`
9. Install `microk8s`

## Install base image

- [Getting started](https://docs.radxa.com/en/rock4/rock4se/getting-started)
- [Ubuntu install guide](https://wiki.radxa.com/Rockpi4/Ubuntu)
- [Ubuntu images](https://github.com/radxa-build/rock-4se/releases), currently using `rock-4se_ubuntu_jammy_cli_b38.img.xz`
- Flash the Ubuntu image using Etcher

## Passwords

```bash
sudo su
passwd rock
passwd root
```

## Enable WiFi

```bash
sudo su

# turn wifi on
nmcli r wifi on

# scan
nmcli dev wifi

# connect
nmcli dev wifi connect "wifi_name" password "wifi_password"
```

## Hostnames

We need to set different hostnames for the different boards.

Run one for each board.

```bash
hostnamectl set-hostname rockpi-4b-m01
hostnamectl set-hostname rockpi-4b-m02
```

## Tools/utils

### Install signing keyring

See [Sudo apt update get error](https://forum.radxa.com/t/sudo-apt-update-get-error/13061/8)

```bash
keyring="$(mktemp)"
version="$(curl -L https://github.com/radxa-pkg/radxa-archive-keyring/releases/latest/download/VERSION)"
curl -L --output "$keyring" "https://github.com/radxa-pkg/radxa-archive-keyring/releases/latest/download/radxa-archive-keyring_${version}\_all.deb"
sudo dpkg -i "$keyring"
rm -f "$keyring"
```

```bash
sudo apt update
sudo apt install less
sudo apt install git
sudo apt install snapd
```

## MicroK8s

_Addon notes:_

- `metrics-server` is used to be able to run commands like `top nodes`
- `dashboard` is not needed imho
- `metallb` is used to give cluster a "public" hostname and ip, pick an unused IP-range, ie 192.168.1.220-192.168.1.220

```bash
# Base install
sudo snap install microk8s --classic
sudo snap install microk8s --classic --channel=1.32/stable

# Add rock user to microk8s group
sudo usermod -a -G microk8s $USER
mkdir -p ~/.kube
chmod 0700 ~/.kube
su - $USER

# Check status
microk8s status --wait-ready

# Addons
microk8s enable metrics-server
microk8s enable dns
microk8s enable ingress
microk8s enable rbac
microk8s enable metallb

# Test
microk8s kubectl create deployment microbot --image=cdkbot/microbot-arm64:latest
microk8s kubectl scale deployment microbot --replicas=3
```

## Nodes

Use the commands from `add-node` to join the cluster from each node (as worker or control plane).

```bash
microk8s add-node
```

### Cluster Admins

```bash
# Generate key
openssl genrsa -out <user>.key 2048

# Create a signing request
openssl req -new -key <user>.key -out <user>.csr -subj "/CN=<user>/O=system:masters"

# Create and sign the certificate with the cluster CA
openssl x509 -req -in <user>.csr -CA /var/snap/microk8s/current/certs/ca.crt -CAkey /var/snap/microk8s/current/certs/ca.key -CAcreateserial -out <user>.crt -days 365 -sha256

# Verify the created certficiate
openssl x509 -in <user>.crt -text -noout

# Assign Role cluster-admin to the user
kubectl create clusterrolebinding <user>-cluster-admin --clusterrole=cluster-admin --user=<user>

# Assign Role admin to the user, seems neccessary for accessing Metrics API
kubectl create clusterrolebinding <user>-admin --clusterrole=admin --user=<user>
```

Run on client

```bash
kubectl config set-cluster rock-cluster --server=https://192.168.1.154:16443 --certificate-authority=./ca.crt --embed-certs=true
kubectl config set-credentials <user> --client-certificate=./user.crt --client-key=./user.key
kubectl config set-context rock --cluster=rock-cluster --user=<user>
```
