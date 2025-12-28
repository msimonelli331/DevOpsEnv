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
   helm template coredns devopsenv/coredns --namespace kube-system --set vars.domain=devops --set vars.ip=127.0.0.1 > /var/snap/microk8s/common/addons/core/addons/dns/coredns.yaml
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

5. **DEPRECATED** Setup TLS for ingresses following steps [here](https://arminreiter.com/2022/01/create-your-own-certificate-authority-ca-using-openssl/). **Note: Change .devops to match your selected domain name**

   ```bash
   openssl genrsa -aes256 -out ca.key 4096

   openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/C=<Country 2 letter code>/ST=<State 2 letter code>/L=<City>/O=DevOpsEnv/CN=DevOpsEnv Root CA"

   openssl req -new -nodes -out server.csr -newkey rsa:4096 -keyout server.key -subj '/CN=DevOpsEnv Server/C=<Country 2 letter code>/ST=<State 2 letter code>/L=<City>/O=DevOpsEnv'

   openssl req -x509 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256 -subj "/CN=devops/O=DevOpsEnv" -addext "subjectAltName = DNS.1:devops,DNS.2:*.devops"

   openssl x509 -in server.crt -text -noout

   kubectl create secret tls ingress-tls -n devops --cert=server.crt --key=server.key
   ```

### Cert Manager

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added

#### Install Steps

1. Install the cert-manager helm chart

   ```bash
   curl -LO https://cert-manager.io/public-keys/cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg

   helm install \
   cert-manager oci://quay.io/jetstack/charts/cert-manager \
   --version v1.19.2 \
   --namespace cert-manager \
   --create-namespace \
   --verify \
   --keyring ./cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg \
   --set crds.enabled=true
   ```

2. Install the certs helm chart

   ```bash
   helm install certs devopsenv/certs --create-namespace -n devops
   ```

### Loadbalancer

#### Preconditions

1. Internet Connection
2. OS with Kubernetes

#### Install Steps

1. Enable the metallb loadbalancer

   ```bash
   microk8s enable metallb
   ```

2. When prompted input an ip range that is in your subnet

3. Get the ip addr of the excordns service, copy the EXTERNAL IP for the next step

   ```bash
   kubectl get svc external-dns -n kube-system
   ```

4. Edit /etc/resolv.conf or a static ip config and paste in the IP from the previous step as the DNS server

Your host should now be able to resolve Ingress hostnames and the internet

### Nexus

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added
4. CoreDNS updated to resolve ingress hostnames
   - A separate hostname for the container repo is required to docker login
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

   ```bash
   helm upgrade nexus devopsenv/nexus -n devops
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

9. Navigate to http://nexus.devops/ and enable the Docker Bearer Token Realm in Nexus Security->Realms Tab

10. Trust the certs we created for the private container registry. **Note: Change container.devops to match your selected host and domain name for the docker repo**

    **DEPRECATED**

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

    **UPDATED**

    ```bash
    kubectl get secret -n devops devops-ca -o jsonpath='{.data.tls\.crt}' | base64 -d > /etc/pki/ca-trust/source/anchors/ca.crt
    update-ca-trust

    mkdir -p /var/snap/microk8s/current/args/certs.d/container.devops
    cat > /var/snap/microk8s/current/args/certs.d/container.devops/hosts.toml << EOF
    server = "https://container.devops"

    [host."https://container.devops"]
    capabilities = ["pull", "resolve"]
    EOF
    ```

11. Restart microk8s

    ```bash
    systemctl start snapd
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

   ```bash
   helm upgrade gitea devopsenv/gitea -n devops
   ```

5. Add the gitea hostname to `etc/hosts`. **Note: Change git.devops to match your selected host and domain name for the gitea server**

   - Append git.devops to the `127.0.0.1 localhost ... git.devops` line
   - Do this on the server machine and your client machine

6. Navigate to http://git.devops/ for initial configuration
   1. Configure the database as sqlite3
   2. Add an admin user
   3. Add the following secrets in the Settings > Actions > Secrets
      - NEXUS_USERNAME
      - NEXUS_PASSWORD
      - OLLAMA_IP

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

4. **DEPRECATED** base64 encode the ca.crt created for the private container registry

   ```bash
   cacert=$(cat /etc/pki/ca-trust/source/anchors/ca.crt | base64 -w 0)
   ```

5. Go to gitea webpage > Site Administration > Actions > Runners > Create new Runner > Copy the token

6. base64 encode the runner token. **Note: You must use `echo -n` otherwise it will encode a newline character**

   ```bash
   token=$(echo -n "<registration token from previous step>" | base64 -w 0)
   ```

7. Install the helm chart, updating the hostname to match the selected domain name from previous steps. **Note: If you used a different domain name and/or host name you have to download and update the values file**

   ```bash
   helm install gitea-runner devopsenv/gitea-runner --create-namespace -n devops \
    --set token=${token} \
    -f gitea-runner-values.yaml
   ```

   ```bash
   helm upgrade gitea-runner devopsenv/gitea-runner -n devops
   ```

**Note: If creating a new runner you must delete the .runner file: rm /mnt/devops/gitea-runner/.runner**

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

1.  Setup a wireguard server on the DevOpsEnv machine

    ```bash
    dnf install wireguard-tools
    sysctl -w net.ipv4.ip_forward=1
    systemctl disable --now firewalld

    wg genkey | tee /etc/wireguard/private.key
    cat /etc/wireguard/private.key | wg pubkey | tee /etc/wireguard/public.key

    cat > /etc/wireguard/wg0.conf << EOF
    [Interface]
    PrivateKey = $(cat /etc/wireguard/private.key)
    Address = 192.168.<subnet>.1/24
    ListenPort = 51820
    SaveConfig = true
    PostUP = /etc/wireguard/postup.sh
    PostDown = /etc/wireguard/postdown.sh
    EOF
    ```

2.  Setup the client config on the same machine

    ```bash
    wg genkey | tee /etc/wireguard/client-private.key
    cat /etc/wireguard/client-private.key | wg pubkey | tee /etc/wireguard/client-public.key

    cat > /etc/wireguard/wg-client.conf << EOF
    [Interface]
    PrivateKey = $(cat /etc/wireguard/client-private.key)
    Address = 192.168.<subnet>.2/24
    DNS = 8.8.8.8

    [Peer]
    PublicKey = $(cat /etc/wireguard/public.key)
    AllowedIPs = 0.0.0.0/0, ::/0
    Endpoint = <devopsenv public ip>:51820
    PersistentKeepalive = 5
    EOF
    ```

3.  Create post up and post down scripts, setting `nic` to the interface you want to forward wireguard traffic through

    - Option 1: Route traffic to a lan

      ```bash
      cat > /etc/wireguard/postup.sh << EOF
      nic=""
      iptables -t nat -A POSTROUTING -o \${nic} -j MASQUERADE
      iptables -A FORWARD -i \${nic} -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
      iptables -A FORWARD -i wg0 -o \${nic} -j ACCEPT

      iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
      iptables -A FORWARD -i wg0 -o \${nic} -m state --state RELATED,ESTABLISHED -j ACCEPT
      iptables -A FORWARD -i \${nic} -o wg0 -j ACCEPT

      wg set wg0 peer $(cat /etc/wireguard/client-public.key) allowed-ips 192.168.<subnet>.2/32
      EOF
      chmod +x /etc/wireguard/postup.sh
      ```

      ```bash
      cat > /etc/wireguard/postdown.sh << EOF
      nic=""
      iptables -t nat -D POSTROUTING -o \${nic} -j MASQUERADE
      iptables -D FORWARD -i \${nic} -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
      iptables -D FORWARD -i wg0 -o \${nic} -j ACCEPT

      iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE
      iptables -D FORWARD -i wg0 -o \${nic} -m state --state RELATED,ESTABLISHED -j ACCEPT
      iptables -D FORWARD -i \${nic} -o wg0 -j ACCEPT
      EOF
      chmod +x /etc/wireguard/postdown.sh
      ```

    - Option 2: Route traffic through the internet connected nic

      ```bash
      cat > /etc/wireguard/postup.sh << EOF
      nic=""
      iptables -A FORWARD -i wg0 -j ACCEPT
      iptables -t nat -A POSTROUTING -o \${nic} -j MASQUERADE

      wg set wg0 peer $(cat /etc/wireguard/client-public.key) allowed-ips 192.168.<subnet>.2/32
      EOF
      chmod +x /etc/wireguard/postup.sh
      ```

      ```bash
      cat > /etc/wireguard/postdown.sh << EOF
      nic=""
      iptables -D FORWARD -i wg0 -j ACCEPT
      iptables -t nat -D POSTROUTING -o \${nic} -j MASQUERADE
      EOF
      chmod +x /etc/wireguard/postdown.sh
      ```

4.  Start the server

    ```bash
    systemctl enable --now wg-quick@wg0.service
    ```

5.  Create a Wireguard client using the files generated above

6.  (Optional) Setup a mobile WG client

    ```bash
    dnf install epel-release -y
    dnf install qrencode
    qrencode -t png -o client-qr.png -r /etc/wireguard/wg-client.conf
    ```

    - Transfer the png to a windows machine, open it and scan it in the wireguard app

## Resources

- RockyLinux
  - https://rockylinux.org/download
  - https://snapcraft.io/docs/installing-snap-on-rocky
- MicroK8s
  - https://microk8s.io/docs/getting-started
  - https://kubernetes.github.io/ingress-nginx/deploy/#microk8s
  - https://microk8s.io/docs/addon-ingress
  - https://stackoverflow.com/questions/55672498/kubernetes-cluster-stuck-on-removing-pv-pvc
- SSL Certs
  - https://arminreiter.com/2022/01/create-your-own-certificate-authority-ca-using-openssl/
- CoreDNS
  - https://stackoverflow.com/questions/55958507/helm-templating-variables-in-values-yaml
  - https://github.com/coredns/helm/tree/coredns-1.24.5/charts/coredns/templates
  - https://github.com/canonical/microk8s-core-addons/blob/a8d3cd9e300d66012c01c1ecf364ddd23657b97c/addons/dns/coredns.yaml
  - https://github.com/k3s-io/k3s/blob/54e3b441477d76f822d42b56fe4e72dc79114b05/manifests/coredns.yaml
  - https://www.reddit.com/r/kubernetes/comments/ferapk/helm_set_namespace_for_subchart_dependencies/
  - https://stackoverflow.com/questions/68559495/helm-dependencies-with-different-namespaces
  - https://github.com/helm/helm/issues/5465
  - https://github.com/helm/helm/issues/3553
  - New
    - https://github.com/coredns/coredns/tree/v1.13.2?tab=readme-ov-file#compilation-from-source
    - https://github.com/coredns/coredns/tree/v1.13.2?tab=readme-ov-file#compilation-with-docker
    - https://coredns.io/explugins/k8s_gateway/
    - https://github.com/ori-edge/k8s_gateway
    - https://github.com/ori-edge/k8s_gateway/blob/master/examples/install-clusterwide.yml
- LoadBalancer
  - https://github.com/metallb/metallb/issues/1154
  - https://medium.com/@muppedaanvesh/deploying-nginx-on-kubernetes-a-quick-guide-04d533414967
  - https://github.com/ori-edge/k8s_gateway/issues/279
  - https://github.com/Joker9944/k8s-config/blob/5f1ede42e9dbb8f12d9e642c6d7a357ead51a0cd/apps/nameserver-apps/blocky/helm-release.yaml#L215-L278
- Nexus
  - https://help.sonatype.com/en/installation-methods.html
  - https://github.com/sonatype/nxrm3-ha-repository/blob/main/nxrm-ha/values.yaml
  - https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes
  - https://community.sonatype.com/t/docker-login-401-unauthorized/1345
- Gitea
  - https://docs.gitea.com/installation/install-with-docker#configure-the-user-inside-gitea-using-environment-variables
- Gitea Runner
  - https://gitea.com/gitea/act_runner/src/branch/main/examples/kubernetes/dind-docker.yaml
  - https://docs.docker.com/engine/security/certificates/
  - https://hub.docker.com/r/gitea/act_runner
  - https://hub.docker.com/_/docker
  - https://forum.gitea.com/t/whats-the-idiomatic-way-of-using-gitea-hosted-container-images-in-actions-jobs/8566
  - https://gitea.com/gitea/act_runner/issues/451
  - https://docs.gitea.com/usage/actions/act-runner
  - https://gitea.com/gitea/act_runner/search?q=forcePull
  - https://github.com/go-gitea/gitea/issues/32029
  - https://forum.gitea.com/t/steps-for-act-runner-with-self-signed-root-ca/10334
- Gitea Actions
  - https://docs.gitea.com/usage/actions/quickstart#use-actions
  - https://docs.gitea.com/usage/actions/act-runner#labels
  - https://forum.gitea.com/t/gitea-actions-run-a-job-from-a-custom-image/8905
  - https://forum.gitea.com/t/using-docker-images-from-private-repository-to-run-actions-in/8571/7
  - https://stackoverflow.com/questions/64033686/how-can-i-use-private-docker-image-in-github-actions
  - https://github.com/vegardit/docker-gitea-act-runner/blob/main/image/config.template.yaml
  - https://gitea.com/gitea/act_runner/issues/329
  - https://gitea.com/gitea/act_runner/src/branch/main/internal/pkg/config/config.example.yaml
  - https://stackoverflow.com/questions/53429486/kubernetes-how-to-define-configmap-built-using-a-file-in-a-yaml
  - https://github.com/docker/setup-buildx-action
  - https://github.com/moby/buildkit/blob/master/docs/buildkitd.toml.md
  - https://github.com/docker/build-push-action
  - https://github.com/docker/setup-buildx-action/issues/177
  - https://github.com/kubernetes/kubernetes/issues/23404#issuecomment-203404986
- Helm
  - https://helm.sh/docs/chart_template_guide/yaml_techniques/#yaml-anchors
  - https://github.com/helm/helm/issues/9558
- VirtualBox
  - https://ciq.com/blog/how-to-install-the-virtualbox-guest-additions-so-your-rocky-linux-vms-with-a-gui-can-benefit-from-screen-resizing/
- Cleaning up images
  - https://discuss.kubernetes.io/t/microk8s-images-prune-utility-for-production-servers/15874
  - https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md
- Wireguard
  - https://www.wireguard.com/quickstart/
  - https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04
  - https://www.cyberciti.biz/faq/how-to-generate-wireguard-qr-code-on-linux-for-mobile/
  - https://wireguard.how/client/ios/
  - https://serverfault.com/questions/1159572/route-all-traffic-through-wireguard
  - https://ubuntu.com/server/docs/common-tasks-in-wireguard-vpn
  - https://www.reddit.com/r/WireGuard/comments/g97zkf/wgquick_and_hot_reloadsync/
  - https://gist.github.com/nealfennimore/92d571db63404e7ddfba660646ceaf0d
  - https://unix.stackexchange.com/questions/735074/wireguard-how-to-route-internet-traffic-through-a-mobile-peer
  - https://documentation.ubuntu.com/server/how-to/wireguard-vpn/troubleshooting/
- Cert Manager
  - https://cert-manager.io/docs/installation/helm/
  - https://cert-manager.io/docs/usage/ingress/
  - https://kubernetes.io/docs/concepts/configuration/secret/

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
