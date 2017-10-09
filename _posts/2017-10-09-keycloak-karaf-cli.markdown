---
layout: post
title:  "More Fuse and Karaf integration with Keycloak"
date:   2017-10-09 14:29:00 -0400
categories: fuse sso karaf x509 smartcard 
---

# Introduction 
In a previous post[^1] we integrated Fuse's web console (Hawtio) with Keycloak for X.509 authentication.  This article builds on the previous configuration to integrate Fuse's command line interface with Keycloak.

# Environment
This exercise was all run locally on my laptop, and all references were set up for localhost.  

* Keycloak 3.2.1.Final
* Fuse 6.3.0 R4
* Fedora 26 with both Fuse and Keycloak installed
* Yubikey Neo with a X.509 certificate issued to the 9A (PIV auth) slot
* Firefox with opensc pkcs11 security module
* OpenSSL test CA - root and intermediate CA for issuing certs[^3]

# JMX/SSH password integration with Keycloak
The first step is relatively easy; the direct access configuration for the Demo realm is added to ${KARAF\_HOME}/etc/keycloak-direct-access.json:

```
{
    "realm": "demo",
    "resource": "ssh-jmx-admin-client",
    "ssl-required" : "external",
    "auth-server-url" : "http://localhost:8080/auth",
    "credentials": {
        "secret": "password"
    }
}
```
The ${KARAF\_HOME}/etc/org.apache.karaf.shell.cfg is edited for:

```
sshRealm = keycloak
```

The ${KARAF\_HOME}/etc/org.apache.karaf.management.cfg is edited for:

```
jmxRealm = keycloak
```

Finally, test login:

```
ssh  -p 8101 -oHostKeyAlgorithms=+ssh-dss  admin@localhost
```

# What about smart card (X.509) authentication to the Karaf CLI?
This is easier said than done.  The SSH server doesn't natively support X.509 authentication, and there is no facility in RH-SSO for SSH public key authentication. One idea I have not tried is to do the public key verification (e.g. with a ssh client configured to use the token[^2]) at the Fuse level and then check the user and roles at the Keycloak level.

# A-MQ and Keycloak
This is similar to setting up ssh and jmx for the direct access keycloak configuration.  Just configure ${KARAF\_HOME}/etc/activemq.xml to use the keycloak jaas realm instead of the default karaf one.  An example would look like:

```
<jaasAuthenticationPlugin configuration="keycloak" />
<authorizationPlugin>
        <map> 
                <authorizationMap groupClass="org.apache.karaf.jaas.boot.principal.RolePrincipal">
                        <authorizationEntries>
                                <authorizationEntry queue=">" read="viewer,admin" write="admin" admin="admin"/>
                                <authorizationEntry topic=">" read="viewer,admin" write="admin" admin="admin"/>
                                <authorizationEntry topic="ActiveMQ.Acvisory.>" read="viewer,admin" write="admin" admin="admin"/>
                        </authorizationEntries>
                        <tempDestinationAuthorizationEntry>
                                <tempDestinationAuthorizationEntry read="viewer,admin" write="admin" admin="admin"/>
                        </tempDestinationAuthorizationEntry>
                </authorizationMap>
        </map>
</authorizationPlugin>
```

# Fuse Fabric and Keycloak
Here is an example of how to enable Keycloak for Hawtio authentication in a Fuse Fabric environment.  This results in the same PIV/X.509 authentication as seen previously[^1].

```
fabric:create --wait-for-provisioning
profile-create mykeycloak
profile-edit --repository mvn:org.keycloak/keycloak-osgi-features/3.2.1.Final/xml/features
profile-edit --repository mvn:org.keycloak/keycloak-osgi-features/3.2.1.Final/xml/features mykeycloak 
profile-edit --feature keycloak mykeycloak 
profile-edit --system hawtio.keycloakEnabled=true mykeycloak 
profile-edit --system hawtio.realm=keycloak mykeycloak 
profile-edit --system hawtio.rolePrincipalClasses=org.keycloak.adapters.jaas.RolePrincipal,org.apache.karaf.jaas.boot.principal.RolePrincipal mykeycloak 
profile-edit --system hawtio.keycloakClientConfig=${karaf.base}/etc/keycloak-hawtio-client.json mykeycloak 
container-add-profile root mykeycloak
```
*N.B. - there is a bug[^4] prior to Red Hat JBoss Fuse 6.3 R4 when referencing the json files in the etc directory.

# References

[^1]:[X.509 Auth to Hawtio](https://sjhiggs.github.io/fuse/sso/x509/smartcard/2017/03/29/fuse-hawtio-keycloak.html)
[^2]:[X.509/PKCS11 and SSH](https://developers.yubico.com/PIV/Guides/SSH_with_PIV_and_PKCS11.html)
[^3]:[OpenSSL CA Instructions](https://jamielinux.com/docs/openssl-certificate-authority/)
[^4]:[ENTESB-6777](https://issues.jboss.org/browse/ENTESB-6777)

