# Logging into OpenShift with Active Directory credentials

The official documentation is [here](https://docs.openshift.com/container-platform/latest/authentication/identity_providers/configuring-ldap-identity-provider.html).
My favorite examples are found [here](https://examples.openshift.pub/cluster-configuration/authentication/activedirectory-ldap/).


OpenShift uses [RFC2255-formatted URLs](https://www.rfc-editor.org/rfc/rfc2255.html) for authentication

ldap:// uses port 389. You must also set `insecure: true`.
`ldap://host.fqdn/basedn?attribute?scope?filter`

ldaps:// uses port 636. You should provide the CA via OpenShift's web UI.
`ldaps://host.fqdn/basedn?attribute?scope?filter`

I've also seen port 3269 used when querying a Global Catalog.
`ldaps://host.fqdn:3269/basedn?attribute?scope?filter`

Use this command to reveal the CA.
`openssl s_client -showcerts -connect host.fqdn:636 < /dev/null`

## Enable debug logging for the authentication Pods

OpenShift does not emit a lot of detail regarding failed logins. Its very easy to increase the verbosity of the authentication pods in order to identify common issues like:

- untrusted certificates when connecting to Active Directory
- bad credentials of the bindDN (Service Account)
- users not being found because of uid / sAMAccountName mismatch

The commands below enable debug logging, aggregate all of the logs from all of the authentication pods, and return the logging level to default / normal when everything is confirmed to be working.

```
# Enable debug logging (wait for new pods to rollout)
oc patch authentications.operator.openshift.io/cluster --type=merge -p '{"spec":{"logLevel":"Debug"}}'

# Follow the logs from all of the authentication pods
oc logs -f -l app=oauth-openshift -n openshift-authentication | grep -v -e healthz -e metrics

# Set the log level back to default / normal
oc patch authentications.operator.openshift.io/cluster --type=merge -p '{"spec":{"logLevel":"Normal"}}'

```

:::warning
It is very common for Active Directory to not have a `uid` attribute in its schema. The default RFC2255 URL uses `uid` to confirm if a user account exists. For Active Directory we need to look for `sAMAccountName` instead by adding `...?sAMAAccountName` to the end of the LDAP URL

For example:

```
uri: ldaps://ad.example.com/CN=Users,DC=example,DC=com?sAMAccountName
```

:::


We can query the Active Directory schema/attributes from the command-line like this, only returning results where the surname (sn) is Call:

baseDN: DC=example,DC=com
bindDN: CN=OpenShift Service Account,CN=Users,DC=example,DC=com

```
ldapsearch -x -W \
  -H ldaps://ad.example.com \  
  -D CN=OpenShift Service Account,CN=Users,DC=example,DC=com \
  -b CN=Users,DC=example,DC=com \
  sn=Call
```

An example YAML looks like this

```
---
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: Active Directory
    type: LDAP
    url: ldaps://ad.example.com/CN=Users,DC=example,DC=com?sAMAccountName
    insecure: false
    mappingMethod: claim
    ldap:
      attributes:
        email:
          - mail
        id:
          - dn
        name:
          - cn
        preferredUsername:
          - sAMAccountName
      bindDN: CN=OpenShift Service Account,CN=Users,DC=example,DC=com
      bindPassword:
        - name: some-secret-name
      ca:
        - name: some-secret-name
```

## Handling nested groups
Some organizations setup groups that have other groups as their members. You can tell Active Directory to squash group membership before returning results by adding this to the URL
More info here - https://ldapwiki.com/wiki/Wiki.jsp?page=LDAP_MATCHING_RULE_IN_CHAIN
The URL below only allows users to login if they belong to the "OpenShift-Users" group

```
url: ldaps://ad.example.com/CN=Users,DC=example,DC=com?sAMAccountName?sub?(memberOf:1.2.840.113556.1.4.1941:=CN=OpenShift-Users,CN=Groups,DC=example,DC=com)'

```

## Appendix

https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax
...to check if a user "user1" is a member of group "group1". You would set the base to the user DN (cn=user1, cn=users, dc=x) and the scope to base, and use the following query.

```
(memberof:1.2.840.113556.1.4.1941:=cn=Group1,OU=groupsOU,DC=x)
```

Similarly, to find all the groups that "user1" is a member of, set the base to the groups container DN; for example (OU=groupsOU, dc=x) and the scope to subtree, and use the following filter.

```
(member:1.2.840.113556.1.4.1941:=cn=user1,cn=users,DC=x)
```
