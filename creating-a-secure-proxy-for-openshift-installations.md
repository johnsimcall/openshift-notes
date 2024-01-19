# Setting up a secure proxy for OpenShift installations

## Overview

Installing OpenShift, by default, requires a direct connection from your servers to the OpenShift content hosted at the Red Hat-managed quay.io website. Installing OpenShift with "network restrictions" is also possible. Examples of network restrictions could be firewall policies, security policies, and/or using a fully disconnected or air-gapped network. Creating a fully disconnected mirror is one way to solve the network restrictions. **But using a proxy is my preferred way to solve deal with network restrictions.**

Using a proxy, instead of creating a mirror, makes it very easy to update the OpenShift platform and add additional features as soon as they're needed. However, when OpenShift is completely disconnected, the ability to add new features and update OpenShift itself depends on those features and updates being available in the mirror.

:::info
Creating a disconnected mirror is [much easier using the **new `mirror-registry` service and `oc-mirror` command**](https://docs.openshift.com/container-platform/4.14/installing/disconnected_install/installing-mirroring-disconnected.html). I highly recommend using those tools instead of the [*former `oc adm release mirror ...` procedure*](https://docs.openshift.com/container-platform/4.14/installing/disconnected_install/installing-mirroring-installation-images.html).
:::

In my experience, many people avoid using a proxy-based installation because they don't already have a proxy setup. The rest of this post will illustrate the few simple commands required to setup, and secure, a proxy service on Linux using [Squid](http://www.squid-cache.org/). We'll secure the proxy by configuring it to only allow connections to certain websites, like quay.io and redhat.com.


## Installing the Squid proxy

Installing [Squid](http://www.squid-cache.org/) on Red Hat Enterprise Linux is done with three commands. The commands will:
1. Install the Squid RPM
2. Create a firewall rule that allows your clients and servers to talk to Squid
2.5. Save the changes made to the firewall
4. Start the Squid service, and make sure it starts after reboots

```
# Install Squid
sudo yum install squid

# Allow connections to Squid through the firewall - port 3128/tcp
firewall-cmd --add-service squid
firewall-cmd --add-service squid --permanent

# Start Squid now and make it startup after reboots
systemctl enable --now squid
```

## Securing your Squid proxy

Squid ships with a default set of security configurations. The Squid configuration file is located at `/etc/squid/squid.conf`. By default, Squid will prevent clients and servers from using your proxy if those clients aren't a part of your network(s). This means that Squid will allow connections from the devices on your network if they use common `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16` IP addresses, but refuse all other connection attempts. Allowing access from other subnets is done by adding your subnet(s) to the `localnet` group of IPs.

[Configuring username and password-based restrictions](https://wiki.squid-cache.org/Features/Authentication) is also possible, but that's outside of the scope of this document.

Squid can **optionally** act as a fancy firewall that allows access to websites by name instead of by IP address. Many websites, quay.io included, are accessible from multiple IP addresses. Those IP addresses often change without notice. Configuring Squid to only allow access to certain websites - websites that are listed in a file - is done by:
1. Creating a new Access Control List (acl) that looks for websites in a file
2. Denying access to everything not listed in the file
3. Allowing access to websites listed in the file


```
# Open the Squid configuration file for editing
sudo -e /etc/squid/squid.conf


# Check that your workstation's IP and OpenShift's IPs are allowed to use the proxy
# Add a line to the bottom of the "acl localnet src ..." section if needed

acl localnet src 123.45.67.0/24   # OpenShift IP addresses


# Optional: Only allow clients & servers to connect with approved sites
# Squid processes it's configuration file from the top-down!
# Add the lines below AFTER the "acl CONNECT method CONNECT" line

acl Approved_Sites dstdomain "/etc/squid/Approved_Sites.txt"
http_access deny !Approved_Sites
http_access allow Approved_Sites
```

## Restricting Squid to approved websites

This section only applies if you added the three optional configuration lines into your Squid configuration. I've seen many system administrators and security auditors breath a sigh of relief knowing that they are in control of the websites their systems are visiting. Allowing access to websites is as simple as adding those websites to the file your new Access Control List points to. The list can even use regular expressions like `*.example.com` to easily allow access to multiple websites.

The list of **required** and *optional* websites for an OpenShift installation can be found below.

[Configuring your firewall for OpenShift Container Platform](https://docs.openshift.com/container-platform/latest/installing/install_config/configuring-firewall.html)
[Applying IP address filtering to Red Hat Container Registries](https://access.redhat.com/articles/3638561)

:::info
Please be aware that Squid must be restarted after making changes to the list of allowed websites. You can restart Squid using the command `systemctl restart squid`. Squid will wait for 30 seconds before restarting to allow client / server connections to exit cleanly.
:::

```
# Open the "Approved_Sites.txt" file that our Access Control List (acl) references
# This file should be protected so that non-administrative users can't change its contents

sudo -e /etc/squid/Approved_Sites.txt

# OpenShift installation resources
.quay.io             # REQUIRED: allows cdn.quay.io, cdn01.quay.io, etc...
.redhat.io           # OPTIONAL: allows registry.redhat.io
.redhat.com          # OPTIONAL: allows sso.redhat.com for authentication
.openshift.com       # OPTIONAL: to download `oc`, `openshift-install`, and .ISO images
vcenter.example.com  # REQUIRED: if you're installing OpenShift with vSphere integrations

# Other helpful resources
registry.k8s.io         # csi-driver-nfs container image is hosted here
.githubusercontent.com  # csi-driver-nfs Helm chart is hosted here
.docker.io              # may be helpful if you want it allowed
.github.com             # OpenShift's update graph and ACM's ClusterImageSets are hosted here

.github.com             # oc-mirror needs codeload.github.com
.googleapis.com         # oc-mirror needs storage.googleapis.com
```

## Configuring OpenShift to use the proxy during the installation

If you're using [Red Hat's web-based Assisted Installer](https://console.redhat.com/openshift/assisted-installer/clusters/~new), there is a prompt for entering Squid details before downloading your customized installation ISO.

If you're not using the Assisted Installer, you tell OpenShift to use your proxy by adding a few lines to the `install-config.yaml`. This is documented in the [Configuring the cluster-wide proxy during installation](https://docs.openshift.com/container-platform/4.14/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-configure-proxy_installing-platform-agnostic) section of the documentation.

:::warning
Please excuse the tangent... Some networks are secured using a transparent proxy. These proxies are also known as break-and-inspect proxies, or a man-in-the-middle proxy. Transparent proxies will often decrypt the connection between you or your OpenShift nodes and the internet resources they use, like quay.io. This can only be done if you and your OpenShift nodes explicitly trust the encryption certificate your transparent proxy uses. If you or your OpenShift nodes don't trust the proxy's encryption certificate, the connection is immediately terminated.
If this situation applies to you, you can tell OpenShift to trust the transparent decrypting proxy by adding its certificate to the `additionalTrustBundle:` section of the `install-config.yaml` **YOU MUST ALSO SET** `additionalTrustBundlePolicy: Always` as well because OpenShift needs to trust the certificate during installation, but the installation isn't aware that a proxy is being used. The `proxy:` section of your `install-config.yaml` is not configured for transparent proxies.
:::


## Testing the proxy

Here are a few ways to use curl and podman/docker to confirm the proxy works:

`curl` always tricks me because it doesn't care about the upper-case `HTTP_PROXY=...` environment variable, but it will respect the lower-case `http_proxy=...` environment variable or command-line arguments. Here are a few ways to use `curl` to confirm your proxy is working.

```
curl --proxy-user username:password --proxy 192.168.0.99:3128 --head redhat.com
curl -U username:password -x 192.168.0.99:3128 -I redhat.com

export http_proxy=http://username:password@192.168.0.99:3128
curl -I redhat.com
```

`podman`/`docker` usually doesn't care about the `HTTP_PROXY=...` environment variable because it's usually pulling from an HTTPS location. So we have to export either `HTTPS_PROXY=...` or `https_proxy=...` instead. Ugh!

```
export HTTPS_PROXY=http://username:password@192.168.0.99:3128
podman pull registry.access.redhat.com/ubi8

export https_proxy=http://username:password@192.168.0.99:3128
podman pull registry.access.redhat.com/ubi8
```

I usually apply brute force like this to workaround these frustrating issues

```
export http_proxy=http://username:password@192.168.0.99:3128
export HTTP_PROXY=$http_proxy HTTPS_PROXY=$http_proxy https_proxy=$http_proxy
curl -I redhat.com
podman pull registry.access.redhat.com/ubi8
```

# Appendix

## /etc/squid/squid.conf - complete example

Here are the contents of my `/etc/squid/squid.conf` and `/etc/squid/Approved_Sites.txt` files my Squid proxy server. This configuration file came from a RHEL 8 RPM. I only added three lines.

```
[jcall@proxy ~]$ sudo cat /etc/squid/squid.conf
# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 0.0.0.1-0.255.255.255  # RFC 1122 "this" network (LAN)
acl localnet src 10.0.0.0/8             # RFC 1918 local private network (LAN)
acl localnet src 100.64.0.0/10          # RFC 6598 shared address space (CGN)
acl localnet src 169.254.0.0/16         # RFC 3927 link-local (directly plugged) machines
acl localnet src 172.16.0.0/12          # RFC 1918 local private network (LAN)
acl localnet src 192.168.0.0/16         # RFC 1918 local private network (LAN)
acl localnet src fc00::/7               # RFC 4193 local private network range
acl localnet src fe80::/10              # RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
acl Safe_ports port 70          # gopher
acl Safe_ports port 210         # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280         # http-mgmt
acl Safe_ports port 488         # gss-http
acl Safe_ports port 591         # filemaker
acl Safe_ports port 777         # multiling http
acl CONNECT method CONNECT

### John Call added these lines ###
acl Approved_Sites dstdomain "/etc/squid/Approved_Sites.txt"
http_access deny !Approved_Sites
http_access allow Approved_Sites

...
<THE REST OF THE FILE WAS UNTOUCHED - LEAVING IT FOR BREVITY>
```

I prefer to list my allowed websites outside of the main `/etc/squid/squid.conf` configuration file.

```
[jcall@proxy ~]$ sudo cat /etc/squid/Approved_Sites.txt
.openshift.com
.quay.io
.redhat.com
.redhat.io
.googleapis.com  # oc-mirror needs storage.googleapis.com
.github.com      # oc-mirror needs codeload.github.com

vcenter.example.com  # to allow automated OpenShift installs
```