tting up a secure proxy for OpenShift installations

## Overview

Put words here

## Installing Squid

If you're running squid on a RHEL bastion/utility server, you can...

```
  # install squid
yum install squid

  # add the config above to /etc/squid/squid.conf (put it below "acl CONNECT method CONNECT")
vi /etc/squid/squid.conf

  # allow squid through the firewall (3128/tcp)
firewall-cmd --add-service squid --permanent
firewall-cmd --reload

  # turn on the squid daemon, and make it startup after reboots
systemctl enable --now squid

  # watch the logs
tail -f /var/log/squid/access.log
```

## Basic Squid configuration

```
## /etc/squid/squid.conf


# check that your workstation IP and the OpenShift Node IPs are allowed to use the proxy
# add them at the END of the "acl localnet src ..." section if they're not already allowed
acl localnet src 123.45.67.0/24


# only allow connections to approved sites
# add the lines below AFTER the "acl CONNECT method CONNECT" line in your config file
acl Approved_Sites dstdomain "/etc/squid/Approved_Sites.txt"
http_access deny !Approved_Sites
http_access allow Approved_Sites
```

## Restricting Squid to authorized destinations

```
## /etc/squid/Approved_Sites.txt

# if you're installing OpenShift with vSphere integrations, put your vCenter FQDN/address in here too
# the OpenShift Machine API Operator will use the defined cluster proxy when creating Worker nodes/VMs
vcenter.example.com

# https://docs.openshift.com/container-platform/4.11/installing/install_config/configuring-firewall.html
# https://access.redhat.com/articles/3638561
.quay.io        # allows cdn.quay.io
.redhat.io      # allows registry.redhat.io
.redhat.com     # allows sso.redhat.com for authentication
.openshift.com  # allows `oc`, `openshift-install`, and .ISO images

# other helpful resources
.github.com      # the OCP update graph and ACM's ClusterImageSets are pulled from here
registry.k8s.io  # used by `nfs-subdir-external-provisioner`
.docker.io       # may be helpful
```

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
