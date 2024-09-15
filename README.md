# DevOpsEnv

Helm charts and steps for a Kuberenetes DevOps Environment

## Dependencies

To use these helm charts you need a kubernetes cluster. For this example we're going to use Rocky Linux and microk8s

### Rocky Linux

#### Preconditions

1. Internet Connection
2. Hypervisor

#### Install Steps

1. Download Rocky Linux v9.4 Boot ISO from [here](https://rockylinux.org/download)
2. Attach the ISO to a new VM or make a bootable USB

   - Suggested VM specs:

     1. 20480 MB of RAM
     2. 4 CPUs
     3. 100 GB of Storage

3. Choose minimal install
4. Create an admin user
5. Once booted, setup the root user password

   ```bash
   sudo su -
   passwd
   ```

6. Disable firewalld, causes routing problems between pods

   ```bash
   systemctl disable --now firewalld
   ```

7. Install snapd by following steps from [here](https://snapcraft.io/docs/installing-snap-on-rocky)
   ```bash
   dnf install epel-release -y
   dnf install snapd -y
   systemctl start snapd.socket
   ln -s /var/lib/snapd/snap /snap
   ```

### MicroK8s

#### Preconditions

1. Internet Connection
2. OS with snapd installed

#### Install Steps

1. Install microk8s with snap, following steps from [here](https://microk8s.io/docs/getting-started)

   ```bash
   snap install microk8s --classic --channel=1.31
   ```

2. Setup permissions to use microk8s command

   ```bash
   usermod -a -G microk8s $USER
   mkdir -p ~/.kube
   chmod 0700 ~/.kube
   su - $USER
   ```

3. Setup alias' to use commands normally

   ```bash
   snap alias microk8s.kubectl kubectl
   snap alias microk8s.helm helm
   ```

4. Add an ingress controller by following steps from [here](https://kubernetes.github.io/ingress-nginx/deploy/#microk8s) and [here](https://microk8s.io/docs/addon-ingress)

   ```bash
   microk8s enable ingress
   ```

### Helm

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm

#### Install Steps

1. Add this helm repo

   ```bash
   helm repo add devopsenv https://msimonelli331.github.io/DevOpsEnv/
   ```

2. Confirm the repo was added and has charts

   ```bash
   helm search repo devopsenv
   ```

### CoreDNS

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added

#### Install Steps

1. Disable microk8s coredns

   ```bash
   microk8s disable dns
   ```

2. Install coredns, be sure to change the IP to your machines IP. **Note: Change devops to match your desired domain name**

   ```bash
   helm template  coredns devopsenv/coredns --namespace kube-system --set vars.domain=devops --set vars.ip=127.0.0.1 > /var/snap/microk8s/common/addons/core/addons/dns/coredns.yaml
   ```

3. Update `/var/snap/microk8s/common/addons/core/addons/dns/coredns.yaml` to add `namespace: kube-system` to all namespaced resources

   - ServiceAccount
   - ConfigMap
   - Service
   - Deployment

4. Reenable microk8s dns

   ```bash
   microk8s enable dns
   ```

5. Setup TLS for ingresses following steps [here](https://arminreiter.com/2022/01/create-your-own-certificate-authority-ca-using-openssl/). **Note: Change .devops to match your selected domain name**

   ```bash
   openssl genrsa -aes256 -out ca.key 4096

   openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/C=<Country 2 letter code>/ST=<State 2 letter code>/L=<City>/O=DevOpsEnv/CN=DevOpsEnv Root CA"

   openssl req -new -nodes -out server.csr -newkey rsa:4096 -keyout server.key -subj '/CN=DevOpsEnv Server/C=<Country 2 letter code>/ST=<State 2 letter code>/L=<City>/O=DevOpsEnv'

   openssl req -x509 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256 -subj "/CN=devops/O=DevOpsEnv" -addext "subjectAltName = DNS.1:devops,DNS.2:*.devops"

   openssl x509 -in server.crt -text -noout

   kubectl create secret tls ingress-tls -n devops --cert=server.crt --key=server.key
   ```

### Nexus

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added
4. CoreDNS updated to resolve ingress hostnames
   - A separate hostname for the cotnainer repo is required to docker login
   - You cannot login to nexus.devops/repository/container
     - docker strips the /'s and just tries to login to nexus.devops
   - So the container repo needs its own hostname
     - It is likely possible to use the service name and port in a pod or the endpoint ip and port on the host, but it is convienent to use the ingress name in and outside pods
5. server cert generated that has a SAN for the ingress domain
   - https is required to docker login to the container repo

#### Install Steps

1. Create the host volume directory

   ```bash
   mkdir -p /mnt/devops/nexus
   ```

2. Label the node this local volume is running on

   ```bash
   kubectl label nodes localhost.localdomain nexus=local
   ```

3. Install the helm chart, updating the hostnames to match the selected domain name from previous steps. **Note: If you used a different domain name and/or host name you have to download and update the values file**

   ```bash
   helm install nexus devopsenv/nexus --create-namespace -n devops \
    -f nexus-values.yaml
   ```

4. Get the default password

   ```bash
   kubectl exec -it -n devops deploy/nexus -- cat /nexus-data/admin.password
   ```

5. Login to nexus with the password from the last step, change the password

6. Create a docker hosted container repo

   1. Enable HTTP and use the port 8082

7. Create the registry credential for this private repo. **Note: Change container.devops to match your selected host and domain name for the docker repo**

   ```bash
   kubectl create secret -n devops docker-registry regcred --docker-server=container.devops --docker-username=$NEXUS_USERNAME --docker-password=$NEXUS_PASSWORD
   ```

8. Add the container repo hostname to `etc/hosts`. **Note: Change container.devops to match your selected host and domain name for the docker repo**

   - Append container.devops to the `127.0.0.1 localhost ... container.devops` line
   - Do this on the server machine and your client machine

9. Trust the certs we created for the private container registry. **Note: Change container.devops to match your selected host and domain name for the docker repo**

   ```bash
   cp ca.crt /etc/pki/ca-trust/source/anchors/
   update-ca-trust

   mkdir -p /var/snap/microk8s/current/args/certs.d/container.devops
   cat > /var/snap/microk8s/current/args/certs.d/container.devops/hosts.toml << EOF
   server = "https://container.devops"

   [host."https://container.devops"]
   capabilities = ["pull", "resolve"]
   EOF
   ```

10. Restart microk8s

    ```bash
    microk8s stop
    microk8s start
    ```

### Gitea

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added

#### Install Steps

1. Create the host volume directory

   ```bash
   mkdir -p /mnt/devops/gitea/{data,config}
   ```

2. Label the node this local volume is running on

   ```bash
   kubectl label nodes localhost.localdomain gitea=local
   ```

3. Update volume permissions

   ```bash
   chown -R 1000:1000 /mnt/devops/gitea
   ```

4. Install the helm chart, updating the hostname to match the selected domain name from previous steps. **Note: If you used a different domain name and/or host name you have to download and update the values file**

   ```bash
   helm install gitea devopsenv/gitea --create-namespace -n devops \
    -f gitea-values.yaml
   ```

5. Navigate to http://git.devops/ for initial configuration
   1. Configure the database as sqlite3
   2. Add an admin user

### Gitea Runner

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added
4. Gitea deployed and running

#### Install Steps

1. Create the host volume directory

   ```bash
   mkdir -p /mnt/devops/gitea-runner
   ```

2. Label the node this local volume is running on

   ```bash
   kubectl label nodes localhost.localdomain gitea-runner=local
   ```

3. Update volume permissions

   ```bash
   chown -R 1000:1000 /mnt/devops/gitea-runner
   ```

4. base64 encode the ca.crt created for the private container registry

   ```bash
   cacert=$(cat ca.crt | base64 -w 0)
   ```

5. Go to gitea webpage > Settings > Actions > Runners > Create new Runner > Copy the token

6. base64 encode the runner token. **Note: You must use `echo -n` otherwise it will encode a newline character**

   ```bash
   token=$(echo -n "<registration token from previous step>" | base64 -w 0)
   ```

7. Install the helm chart, updating the hostname to match the selected domain name from previous steps. **Note: If you used a different domain name and/or host name you have to download and update the values file**

   ```bash
   helm install gitea-runner devopsenv/gitea-runner --create-namespace -n devops \
    --set token=${token} \
    --set cacert=${cacert} \
    -f gitea-runner-values.yaml
   ```

### Buildkitd

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added

#### Install Steps

4. Install the helm chart

   ```bash
   helm install buildkitd devopsenv/buildkitd --create-namespace -n devops
   ```

### Builder

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. Gitea deployed and running
4. Gitea runner deployed, running, and connected to gitea

#### Install Steps

1. Create a Buidler git repo in gitea

2. Add the contents of the builder folder

3. Follow the steps in the builder/README.md

### Wireguard VPN

#### Preconditions

1. Internet Connection

#### Install Steps

1. Setup a wireguard server on the DevOpsEnv machine

   ```bash
   dnf install wireguard-tools

   wg genkey | tee /etc/wireguard/private.key
   cat /etc/wireguard/private.key | wg pubkey | tee /etc/wireguard/public.key

   cat > /etc/wireguard/wg0.conf << EOF
   [Interface]
   PrivateKey = $(cat /etc/wireguard/private.key)
   Address = 192.168.<subnet>.1/24
   ListenPort = 51820
   SaveConfig = true
   EOF
   ```

2. Setup the client config on the same machine

   ```bash
   wg genkey | tee /etc/wireguard/client-private.key
   cat /etc/wireguard/client-private.key | wg pubkey | tee /etc/wireguard/client-public.key

   cat > /etc/wireguard/wg-client.conf << EOF
   [Interface]
   PrivateKey = $(cat /etc/wireguard/client-private.key)
   Address = 192.168.<subnet>.2/24

   [Peer]
   PublicKey = $(cat /etc/wireguard/public.key)
   AllowedIPs = 192.168.<subnet>.0/24
   Endpoint = <devopsenv public ip>:51820
   EOF
   ```

3. Start the server

   ```bash
   wg set wg0 peer $(cat /etc/wireguard/client-public.key) allowed-ips 192.168.<subnet>.2

   systemctl enable --now wg-quick@wg0.service
   ```

4. Create a Wireguard client using the files generated above

5. (Optional) Setup a mobile WG client

   ```bash
   dnf install qrencode
   qrencode -t png -o client-qr.png -r /etc/wireguard/wg-client.conf
   ```

   - Transfer the png to a windows machine, open it and scan it in the wireguard app

## Resources

- https://rockylinux.org/download
- https://snapcraft.io/docs/installing-snap-on-rocky
- https://microk8s.io/docs/getting-started
- https://kubernetes.github.io/ingress-nginx/deploy/#microk8s
- https://microk8s.io/docs/addon-ingress
- https://arminreiter.com/2022/01/create-your-own-certificate-authority-ca-using-openssl/
- https://stackoverflow.com/questions/55958507/helm-templating-variables-in-values-yaml
- https://github.com/coredns/helm/tree/coredns-1.24.5/charts/coredns/templates
- https://github.com/canonical/microk8s-core-addons/blob/a8d3cd9e300d66012c01c1ecf364ddd23657b97c/addons/dns/coredns.yaml
- https://github.com/k3s-io/k3s/blob/54e3b441477d76f822d42b56fe4e72dc79114b05/manifests/coredns.yaml
- https://help.sonatype.com/en/installation-methods.html
- https://github.com/sonatype/nxrm3-ha-repository/blob/main/nxrm-ha/values.yaml
- https://www.reddit.com/r/kubernetes/comments/ferapk/helm_set_namespace_for_subchart_dependencies/
- https://stackoverflow.com/questions/68559495/helm-dependencies-with-different-namespaces
- https://github.com/helm/helm/issues/5465
- https://github.com/helm/helm/issues/3553
- https://docs.gitea.com/installation/install-with-docker#configure-the-user-inside-gitea-using-environment-variables
- https://discuss.kubernetes.io/t/microk8s-images-prune-utility-for-production-servers/15874
- https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md
- https://gitea.com/gitea/act_runner/src/branch/main/examples/kubernetes/dind-docker.yaml
- https://hub.docker.com/r/gitea/act_runner
- https://hub.docker.com/_/docker
- https://docs.gitea.com/usage/actions/quickstart#use-actions
- https://docs.gitea.com/usage/actions/act-runner#labels
- https://forum.gitea.com/t/gitea-actions-run-a-job-from-a-custom-image/8905
- https://forum.gitea.com/t/using-docker-images-from-private-repository-to-run-actions-in/8571/7
- https://github.com/vegardit/docker-gitea-act-runner/blob/main/image/config.template.yaml
- https://gitea.com/gitea/act_runner/issues/329
- https://gitea.com/gitea/act_runner/src/branch/main/internal/pkg/config/config.example.yaml
- https://stackoverflow.com/questions/53429486/kubernetes-how-to-define-configmap-built-using-a-file-in-a-yaml
- https://ciq.com/blog/how-to-install-the-virtualbox-guest-additions-so-your-rocky-linux-vms-with-a-gui-can-benefit-from-screen-resizing/
- https://www.wireguard.com/quickstart/
- https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04
- https://www.cyberciti.biz/faq/how-to-generate-wireguard-qr-code-on-linux-for-mobile/
- https://wireguard.how/client/ios/

## Notes

### Cleanup old images

```bash
VERSION="v1.30.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
crictl -r unix:///var/snap/microk8s/common/run/containerd.sock rmi --prune
```

### VBoxGuestAdditions

- After adding the ISO to the VM run:

  ```bash
  mkdir -p /mnt/cdrom
  mount /dev/sr0 /mnt/cdrom
  dnf install -y bzip2
  /mnt/cdrom/VBoxLinuxAdditions.run
  umount /mnt/cdrom
  rm -rf /mnt/cdrom
  reboot
  ```
