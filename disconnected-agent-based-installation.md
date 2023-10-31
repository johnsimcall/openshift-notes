# Installing OpenShift in a disconnected network, step-by-step

Making an offline installation bundle for OpenShift requires [mirroring/downloading the container images](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-disconnected.html) and then [hosting those container images in a container registry](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-creating-registry.html) that is accessible by the cluster nodes. The download process can put the container images into the local filesystem or upload them directly into the container registry. (a USB stick, a directory that will be burnt to a DVD, or a folder that will be uploaded into S3 or similar storage)

:::info
A minimal download of OpenShift 4.12 requires ~15GB of space
A minimal download of OpenShift Platform Plus requires ~50GB of space

If you're using mounted storage, consider setting `--quayRoot` to a subdirectory of the mountpoint. Uninstalling the mirror registry will fail otherwise.
:::

Lots of good information in this blog - [Mirroring OpenShift Registries: The Easy Way by Ben Schmaus and Daniel Messer (August 23, 2022)](https://cloud.redhat.com/blog/mirroring-openshift-registries-the-easy-way).

## Local DNS with `dnsmasq` (optional)

`/etc/dnsmasq.d/disconnected.conf`:
```=
interface=eth1
bind-interfaces
# server=10.0.0.1  # Use an upstream DNS server after ours

dhcp-range=192.168.0.180,192.168.0.199
dhcp-option=option:router,192.168.0.10
dhcp-option=option:ntp-server,192.168.0.10

auth-zone=airgap.local
host-record=api.cluster.airgap.local,192.168.0.100
host-record=api-int.cluster.airgap.local,192.168.0.100
host-record=ingress.cluster.airgap.local,192.168.0.101
cname=*.apps.cluster.airgap.local,ingress.cluster.airgap.local
```

## Install `mirror-registry` (aka mini Quay)

```
df -h /data
sudo setfacl -m u:$USER:rwx /data

wget https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz

tar xvzf mirror-registry.tar.gz

./mirror-registry install --help

### password must be at least 8 characters and contain no whitespace
./mirror-registry install --quayRoot /data/mirror-registry --initUser admin --initPassword redhat123

sudo cp -v /data/mirror-registry/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

podman login -u admin -p redhat123 $(hostname -f):8443

firewall-cmd --add-port 8443/tcp --permanent
firewall-cmd --reload
```

### Starting and stopping the `mirror-registry`

Running the `./mirror-registry install ...` results in several `systemd` services being created. You can see which services were created like this:

```
systemctl -a | grep quay
  quay-app.service         loaded    active   running   Quay Container
  quay-pod.service         loaded    active   exited    Infra Container for Quay
  quay-postgres.service    loaded    active   running   PostgreSQL Podman Container for Quay
  quay-redis.service       loaded    active   running   Redis Podman Container for Quay
```

These services will automatically start when the system is rebooted. The `quay-redis`, `quay-db`, and `quay-app` services depend on the `quay-pod` service. You can restart everything with one command:

```
systemctl restart quay-pod
```

### Alternative to `mirror-registry`: Use docker registry

First thing we need to do is create some directories for our container registry we will be setting up and changing the owner to our current user
```
sudo mkdir -p /opt/registry/{auth,certs,data}
sudo chown -R $USER /opt/registry
```

Next we will create a certificate for our registry to use
```
cd /opt/registry/certs
openssl req -newkey rsa:4096 -nodes -sha256 -x509 -days 365  \
  -keyout domain.key -out domain.crt \
  -addext "subjectAltName = DNS:registry.airgap.local" 

Country Name (2 letter code) [XX]:US
State or Province Name (full name) []: North Carolina
Locality Name (eg, city) [Default City]:Raleigh
Organization Name (eg, company) [Default Company Ltd]:Red Hat
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:registry.airgap.local
Email Address []:<your-email-address>test@example.com
```

The common name is the one that matters the rest of these can be pretty much any value but the common name must be the correct name for your machine in order for the certificate to properly resolve

Next we will add simple password authentication on our registry we will just use the username openshift and the password redhat for demonstration purposes
```
htpasswd -bBc /opt/registry/auth/htpasswd openshift redhat
```

Now we can setup our resgistry to run, use the password and certificate we created and automaticatlly start in case the vm ever restarts
```
podman run -d --name mirror-registry \
-p 5000:5000 --restart=always \
-v /opt/registry/data:/var/lib/registry:z \
-v /opt/registry/auth:/auth:z \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
-v /opt/registry/certs:/certs:z \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
docker.io/library/registry:2
```

Update system trust store
```
sudo cp /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

curl -u openshift:redhat https://registry.airgap.local:5000/v2/_catalog
```

## Install the `oc-mirror` plugin

```
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/oc-mirror.tar.gz

mkdir -p $HOME/.local/bin
tar xvzf ./oc-mirror.tar.gz -C $HOME/.local/bin
chmod a+x $HOME/.local/bin/oc-mirror

oc plugin list
oc mirror --help
```

## Add mirror-registry credentials to pull-secret

Download your "pull secret" from the Red Hat OpenShift Cluster Manager at https://console.redhat.com/openshift/install/pull-secret.

```
jq . pull-secret.txt
mkdir -p $HOME/.docker
mv -v pull-secret.txt $HOME/.docker/config.json
podman login -u openshift -p redhat --authfile $HOME/.docker/config.json $(hostname -f):8443

# OPTIONAL - remove the Insights & Telemetry connection
# https://docs.openshift.com/container-platform/4.12/support/remote_health_monitoring/opting-out-of-remote-health-reporting.html
 podman logout --authfile $HOME/.docker/config.json cloud.openshift.com

# Confirm contents and create a backup
jq . $HOME/.docker/config.json
cp -v $HOME/.docker/config.json ~/pull-secret.json
```

```
$ oc-mirror list releases
$ oc-mirror list releases --version 4.12 --channels

$ oc-mirror list operators
$ oc-mirror list operators --version 4.12 --catalogs
$ oc-mirror list operators --version 4.12 \
    --catalog registry.redhat.io/redhat/redhat-operator-index:v4.12 \
    --package odf-operator \
    --channel stable-4.12

Catalog -> "redhat", "certified", "marketplace", or "community"
  Package -> "3scale-operator" | "advanced-cluster-management" ... "web-terminal"
    Channel -> "stable-4.10 | stable-4.11"
      Version -> "4.11.0 | 4.11.1 | 4.11.2"


$ oc-mirror init | tee imageset-config.yaml
$ vi imageset-config.yaml

```

EXAMPLE IMAGESETCONFIG
https://gist.github.com/johnsimcall/27a7bb96a76ee8021c13bc4e2c4ad9fa - NVIDIA
https://gist.github.com/johnsimcall/e27549db5da5cd26206fd0e4fd6b1a61 - OCP-Virt and ODF

## Add `imageContentSources:` to install-config.yaml

oc-mirror, from what I can tell, assumes that you already have a cluster installed because it outputs `ImageContentSourcePolicy` YAML, but unlike `oc adm release mirror` it does not output anything that can be added directly to the `install-config.yaml`.

You can copy-paste from the YAML files. You may have multiple YAML sections, one for `release-0`, another for `operator-0`, and one for `generic-0`. Installation *requires* the `release-0` section, which when added to my `install-config.yaml` looks like this...

```=
apiVersion: v1
baseDomain: example.redhat.com
...
additionalTrustBundlePolicy: Always
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MII...lots of text...
  -----END CERTIFICATE-----
imageContentSources: 
- mirrors:
  - jcall-testing.dota-lab.iad.redhat.com:8443/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - jcall-testing.dota-lab.iad.redhat.com:8443/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
```

## Create the installation files (agent-based)

I'll describe using the new agent-based installer here

### Sample agent-config.yaml

```yaml=
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: cluster
rendezvousIP: 172.31.255.252
hosts:
  - hostname: master-0
    interfaces:
      - name: eno1
        macAddress: 00:50:56:82:8c:dc
    rootDeviceHints:
      deviceName: /dev/sda
  - hostname: master-1
    interfaces:
      - name: eno1
        macAddress: 00:50:56:82:8c:dd
    rootDeviceHints:
      deviceName: /dev/sda
  - hostname: master-2
    interfaces:
      - name: eno1
        macAddress: 00:50:56:82:8c:de
      - name: eno2
        macAddress: 00:50:56:82:8c:df
    networkConfig:
      interfaces:
        - name: eno2
          type: ethernet
          state: down
          mac-address: 00:50:56:82:8c:df
          ipv4:
            enabled: false
```

### Make the agent-based ISO image and boot

```
mkdir ocp-airgap/
cp install-config.yaml.backup ocp-airgap/install-config.yaml

openshift-install agent create agent-config-template
# cp agent-config.yaml.backup ocp-airgap/agent-config.yaml

openshift-install --dir=ocp-airgap agent create image

cd ocp-airgap/
ln -s agent.x86_64.iso ocp-airgap-agent.iso
cd ..

openshift-install --dir=ocp-airgap agent wait-for bootstrap-complete --log-level=debug
openshift-install --dir=ocp-airgap agent wait-for install-complete --log-level=debug
```

## Day 2 Operations

### Disable the default OperatorHub Catalog Sources

```
oc patch OperatorHub cluster --type merge \
  --patch '{"spec":{"disableAllDefaultSources":true}}'
```

### Create a new disconnected Operator Catalog

```
oc get catalogsource --all-namespaces
No resources found

oc create -f oc-mirror-workspace/results-1676565189/catalogSource-redhat-operator-index.yaml
catalogsource.operators.coreos.com/redhat-operator-index created

# oc create -f - <<< $(sed 's/name: redhat-operator-index/name: disconnected-redhat-operators/' oc-mirror-workspace/results-1676565189/catalogSource-redhat-operator-index.yaml)
# catalogsource.operators.coreos.com/disconnected-redhat-operators created

oc get catalogsource --all-namespaces
NAMESPACE               NAME                    DISPLAY   TYPE   PUBLISHER   AGE
openshift-marketplace   redhat-operator-index             grpc               4s

oc get pods -n openshift-marketplace
NAME                                    READY   STATUS    RESTARTS   AGE
marketplace-operator-645774fdc7-hnjgk   1/1     Running   0          20m
redhat-operator-index-sj6q7             1/1     Running   0          44s

oc logs -n openshift-marketplace redhat-operator-index-sj6q7
time="2023-02-16T17:10:14Z" level=info msg="serving registry" configs=/configs port=50051
```

### Disable Automatic Boot Sources for OpenShift Virtualization (if installed)

```bash
$ oc patch hco kubevirt-hyperconverged -n openshift-cnv --type json \
  -p '[{"op": "replace", 
      "path": "/spec/featureGates/enableCommonBootImageImport",
      "value": false}]'
```

### Update Content: Run the mirror process (to a filesystem)

I had trouble mirroring the kubevirt-operator because of a missing `virtio-win` container image. I also had to specify both the "stable" and "stable-1.1" channels of `redhat-oadp-operator` because the dependecy resolution wouldn't proceed with only "stable".

```=
---
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: /data/oc-mirror-imageset-openshift-platform-plus-smaller
mirror:
  platform:
    channels:
    - name: stable-4.12
      type: ocp
  additionalImages:
  - name: registry.redhat.io/ubi8/ubi:latest
  helm: {}
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.12
    packages:
    - name: advanced-cluster-management
      channels:
      - name: release-2.7
    - name: cincinnati-operator
    - name: cluster-logging
      channels:
      - name: stable
    - name: compliance-operator
      channels:
      - name: release-0.1
    - name: devworkspace-operator
      channels:
      - name: fast
    - name: mta-operator
    - name: mtc-operator
    - name: mtr-operator
    - name: odf-operator
      channels:
      - name: stable-4.12
    - name: quay-bridge-operator
      channels:
      - name: stable-3.8
    - name: quay-operator
      channels:
      - name: stable-3.8
    - name: redhat-oadp-operator
      channels:
      - name: stable
      - name: stable-1.1
    - name: rhacs-operator
      channels:
      - name: latest
    - name: web-terminal
```

```
$ grep path imageset-config-openshift-platform-plus-smaller.yaml 
    path: /data/oc-mirror-imageset-openshift-platform-plus-smaller
    
$ time oc mirror file:///data/oc-mirror-imageset-openshift-platform-plus-smaller \
    --config=imageset-config-openshift-platform-plus-smaller.yaml 2>&1 \
      | tee -a imageset-config-openshift-platform-plus-smaller.logs
      





$ time oc mirror file:///data/oc-mirror-imageset-openshift-platform-plus/ \
    --config=imageset-config.yaml.all-operators 2>&1 \
      | tee -a imageset-config.yaml.all-operators.filepath.logs
```

## Run the mirror process (into a Container Registry)

```
$ time oc mirror docker://$(hostname -f):8443 \
    --config=imageset-config.yaml.all-operators 2>&1 \
      | tee -a imageset-config.yaml.all-operators.filepath.logs
```


## Podman tips and tricks

### Manually explore an Operator Catalog (e.g. Red Hat, Community, Certified)

```
# podman run --rm -it --entrypoint=/bin/sh registry.redhat.io/redhat/redhat-operator-index:v4.12

## it's better to "image mount" because "run" the image doesn't provide `jq` or `yq`
podman pull registry.redhat.io/redhat/redhat-operator-index:v4.12
podman unshare
cd $(podman image mount registry.redhat.io/redhat/redhat-operator-index:v4.12)
ls configs
```

```
for i in $(jq --raw-output '. | select( .schema | contains("olm.package")) | .name' configs/*/catalog.json); do echo "    - name: $i" | tee -a ~/imageset-config.yaml.all-operators; done
```

```
jq .name configs/*/catalog.json

# list available operators
# oc mirror list operators --version 4.12 --catalog registry.redhat.io/redhat/redhat-operator-index:v4.12
# the `oc mirror` command takes 60 seconds to run
ls configs/

# browse the metadata of an Operator
jq . configs/advanced-cluster-management/catalog.json | less -i

# report the available channels
# oc mirror list operators --version 4.12 --catalog registry.redhat.io/redhat/redhat-operator-index:v4.12   --package odf-operator     --channel stable-4.12
jq '.schema, .name' configs/advanced-cluster-management/catalog.json 
"olm.package"
"advanced-cluster-management"
"olm.channel"
"release-2.6"
"olm.channel"
"release-2.7"
```

## Poking the Quay API

I wanted to find a way to delete all of the Quay images and start over, without changing my CA certificate, or doing `./mirror-registry uninstall ...` I found that I could create a Quay "super user" token and use that on the command line. [Apparently tokens are deprecated](https://docs.quay.io/glossary/access-token.html), but I couldn't figure out how to make a Robot Account work for me. First create a new Organization. I called mine "adminorg", then [follow the instructions to create a token](https://access.redhat.com/documentation/en-us/red_hat_quay/3/html/red_hat_quay_api_guide/using_the_red_hat_quay_api#create_oauth_access_token).

```
# my TOKEN is 40 characters - a robot account's password/"token" is 64 characters
export TOKEN="abcdefghijklmnopqrstuvwxyz0123456789ABCD"

# sad that this query doesn't list repos as "org/repo"
curl -s -X GET -H "Authorization: Bearer $TOKEN" 'https://jcall-testing.dota-lab.iad.redhat.com:8443/api/v1/repository?public=true' | jq --raw-output '.repositories[].name' 

# must delete "org/repo" instead of "repo"
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" 'https://jcall-testing.dota-lab.iad.redhat.com:8443/api/v1/repository/advanced-cluster-security/rhacs-operator-bundle'

# list all of the organizations, except the "adminorg" which holds my $TOKEN
curl -s -X GET -H "Authorization: Bearer $TOKEN" 'https://jcall-testing.dota-lab.iad.redhat.com:8443/api/v1/superuser/organizations/' | jq --raw-output '.organizations[].name' | grep -v adminorg

# deleting the org removes all repos
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "https://jcall-testing.dota-lab.iad.redhat.com:8443/api/v1/superuser/organizations/advanced-cluster-security"

# delete all of the orgs -- DANGER!!!
for ORG in $(curl -s -X GET -H "Authorization: Bearer $TOKEN" 'https://jcall-testing.dota-lab.iad.redhat.com:8443/api/v1/superuser/organizations/' | jq --raw-output '.organizations[].name' | grep -v adminorg ); do
  echo "Deleting \"$ORG\" organization..."
  curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "https://jcall-testing.dota-lab.iad.redhat.com:8443/api/v1/superuser/organizations/$ORG"
done
```