[![CLA assistant](https://cla-assistant.io/readme/badge/SAP/ipsec-release)](https://cla-assistant.io/SAP/ipsec-release)

Note: This is a heavily modified fork of the project git repository
(https://github.com/CloudCredo/bosh-ipsec). It has been modified and extended
to support an IPsec Cloud Foundry deployment.

# Introduction

The goal of this release is to provide means to secure communication between 
nodes that make up a Cloud Foundry system. This will protect Cloud Foundry 
internal communication as well as application traffic from eavesdroppers. We
have designed the release in a way to provide protection from eavesdroppers
even if parts of the system have been compromised. From an operational point 
of view secure comminication can be enabled and disabled via a Cloud Foundry 
update deployment without causing a downtime of the system.

This release uses IPsec in order to provide transport level encryption. We will
give a brief introduction to IPsec which is needed to carry out the 
configuration steps. For further information refer to the relevant Internet RFCs 
and Racoon documentation which are referenced below.

This bosh release used the IPsec tools \[3\] for maintaining the SPD and the 
IKE key management daemon.

# A Very Short Introduction to IPsec

The Internet RFC 4303 \[1\] specifies a transport mode and a tunnel mode in order 
to ensure the integrity as well as the confidentiality of an IP packet. This
release uses the transport mode as the tunnel mode is usually used for setting
up virtual private networks.

For configuring the system it is important to understand the following concepts:

## Security Policy Database (SPD)

The SPD defines whether and how communication between two peers is to be 
secured. When a package is to be sent to a remote host the kernel checks this
database in order to determine whehter to secure the package.

This is an example dump of SPD record from a test system:

    192.168.33.11[any] 192.168.33.12[any] 255
        in prio def ipsec
        esp/transport//use
        created: Dec 14 16:23:15 2015  lastused:
        lifetime: 0(s) validtime: 0(s)
        spid=544 seq=2 pid=2797
        refcnt=1
    192.168.33.12[any] 192.168.33.11[any] 255
        out prio def ipsec
        esp/transport//use
        created: Dec 14 16:23:15 2015  lastused:
        lifetime: 0(s) validtime: 0(s)
        spid=537 seq=3 pid=2797
        refcnt=1

This means that all communication from 192.168.33.11 to 192.168.33.12 (any port)
and vice versa will be secured by ESP. In addition to IP addresses also CIDRs 
can be specified, e.g. `192.168.33.0/24` in order to signal that all traffic to 
and from a particular network is to be secured. The SPD is rather static and 
usually does not change.

Policies are uni-directional. For full duplex operation a separate policy must 
be defined for both directions as shown in the example above.

## Security Association (SA) and Security Association Database (SAD)

Before being able to communicate via IPsec it is necessary to establish a 
security association (SA) between peers. This association usually contains 
the encryption keys. Once the kernel has checked the SPD and determined that 
the package must be secured it looks for the key in the SAD.
Security associations are maintained in the SAD.

## Security Parameter Index (SPI)

IPsec (ESP) packets contain an index into the SAD, named the SPI. This is needed
by target peer in order to validate, authenticate, and/or decrypt the message.

## The Internet Key Exchange (IKE)

Security associations are maintained in the SAD. Security associations can 
either be configured manually or automatically using the Internet Key 
Exchange Protocol (IKE) (Internet RFC 2409) \[2\]. Static 
configuration for a dynamic landscape is rather difficult and cannot be 
considered secure. This release uses a mechanism based on the IKE protocol 
in order to establish mutual authentication and maintain security associations 
when needed. Initial trust between peers is established using certificates.


# Setting up Cloud Foundry with IPsec

This release is based on the ipsec-tools project \[3\]. It is specifically 
based on two components from the project:

* the ```setkey``` utility to set up kernel IPSEC policies, specifically the SPD.
* the ```racoon``` IKE daemon in order to negotiate security associations

The IPsec job is a co-deployment and must be deployed on all virtual machines 
which take part in the IPsec mesh. The following template in a job definition 
will do:

```
  - name: racoon
    release: ipsec
```

The following properties must be configured on a global level as they are 
needed by every job:

```
properties:
  racoon:
    certificate_authority_cert: |+
      -----BEGIN CERTIFICATE-----
      [ certificate goes here]
      -----END CERTIFICATE-----
    certificate_authority_private_key: |+
      -----BEGIN PRIVATE KEY-----
      [ private key goes here]
      -----END PRIVATE KEY----- 
```

See below for instructions on how to create a certificate authority.

Each job requires a separate configuration for ipsec in the properties section;
this is an excerpt for the consul job:

```
- name: consul_z1
  instances: 1
  jobs:
  - name: racoon
    release: ipsec
    properties:
      ports:
      - name: cf1
        targets:
        - 10.1.1.0/24
  - name: consul_agent
    release: cf
  - name: metron_agent
    release: cf
  networks:
  - name: cf1
    static_ips:
    - 10.1.1.0
```

This means that all outgoing and incoming traffic from network
10.1.1.0/24 shall be secured. There can be multiple networks. The name property 
specifies the network to be used. This is required as there can be multiple 
networks.

In addition to the properties described here there are more properties which
are described in the spec file.

# Operational Considerations

## Activating IPsec without Downtime 

When enabling or disabling IPsec (set the disabled property to 'false' 
respective 'true') Cloud Foundry will hang during the update deployment and the
deployment will eventually fail. The problem is that by default jobs will 
be re-deployed with IPsec on or off, meaning they cannot talk to nodes anymore
with the opposite setting. One common failure (but probably not the only
failure) is that runner nodes cannot comunicate with NATS anymore and thus
cannot report back when the VM is drained. 

The solution is a two stage approach which I will describe by example of 
turning IPsec on but the same will work for turning it off as well. First set
the disabled property to 'false' and add the level property to 'use' in the
racoon global properties:

```
  disabled: false
  level: use
```

The `use` parameter instructs IPsec to use an encrypted connection if one can
be established but still ensures that unencrypted connections continue to work.
This means that the failure as described above will not occur during deployment.
The system must be redeployed with these parameters.

Secondly after successful deployment with the level property set to `use` the
level propery can be set to `require`. This means that only encrypted
connections are  possible. You need to re-deploy the system again. After
the deployment all connections that were configured to be seucre are secured.

*Disclaimers:* 

- The descrived procedure worked fine for our landscape. Do not try productively
  unless you have tested that it works for your landscape.

- Do not operate your landscape with the setting `level: 'use'`. This is a 
  set-up that is not secure.

## Expiration of Certificates

There are two types of certifictes used in the landscape:

1. certificate for the private keys on each node singed by the certificate 
   authority
2. certificate authority certificate 

An expiration of a certificate causes a downtime of the whole system or part 
of it. The possiblity that (1) expires is relatively low as the private key 
and the certificate is created on each deploy operation.

An expiration of the certifcate authority certificate may happen depending on 
the validity that has been specified. The certificate authority certificate 
can be exchanged by setting the `level` property to `use`, configuring the 
new certificate (and possibly a new private key) in the deployment descriptor
and re-deploying the landscape. After that the `level` setting should be set to
`require` before re-deploying again. See also section "Activating IPsec without
Downtime" above.

# Security Considerations

This section discusses the level of security and possible attack scenarios for
the solution provided by this release.

The default configuration in the spec file regarding the negotiation of 
security associations and regarding the encryption of traffic can be considered
secure as of today.

IPsec is widely used mainly for setting up and securing virtual private
networks and can generally be considered secure with rather conservative
default settings as used by this release. The only known issue has been mentioned
on Wikipedia however with very little detail:

> Leaked NSA presentations released by 'Der Spiegel' indicate that ISAKMP is
> being exploited in an unknown manner to decrypt IPSec traffic, as is IKE.[4]

Trust between nodes is established using public key cryptography using
certificates. Each node maintains its own private key and certificate which is 
re-created on each deployment and the cerfifcate is issued using the certificate
authority.

The default setup should be relatively secure, however there are some attack
scenarios:

- If an attacker is able to log on a virtual machine the attacker can read 
  all incoming and outgoing communication. The attacker can however not 
  read traffic between other hosts. 

- There is no single secret which would allow an attacker to decrypt all 
  prior communication at a later point in time and which may have been 
  recorded.

- Security associations are regularly re-negotiated between nodes. All nodes use 
  different keys in order to communicate with each other. Even if an attacker
  is able to get knowledge of a session key it will be short lived and only 
  valid for one single security assocation.

- Knowledge of the private key of the certificte authority would allow man 
  of the middle attacks during connection establishment. An effort should be 
  taken in order to keep the private key secure. In that context an extension
  to bosh has been discussed in order to move the certificate authority to 
  bosh and to reduce the risk of leaking the private key.

The current implementation is based on racoon which to the best of our 
knowledge does not have known security issues, however it is being criticised 
because the IKE daemon is running with `root` privileges. Newer implmentation 
such as Openswan do not have this possible security issue.

# Outlook

There are several aspects which should be addressed in future releases:

- Maintenance of the SAD should be done through the ip command which is
  standard in linux distributions
- There has been some criticism towards the ipsec tools \[5\]. There are more 
  modern implementations that should be used and that support IKE v2 (e.g. 
  strongSwan).
- The racoon job should be added as an addon in order to simplify the 
  configuration and reduce the effort

# 3rd Party Components

This BOSH releases includes the following 3rd party components:

- ipsec-tools release 0.8.2 (https://sourceforge.net/projects/ipsec-tools/files/ipsec-tools/0.8.2/ipsec-tools-0.8.2.tar.bz2/download), see http://ipsec-tools.sourceforge.net/ for details around original authors, licensing, etc.

# References

\[1\] [RFC 4303 - IP Encapsulating Security Payload (ESP)](https://tools.ietf.org/html/rfc4303)

\[2\] [RFC 2409 - The Internet Key Exchange (IKE)](https://tools.ietf.org/html/rfc2409)

\[3\] [IPsec Tools Homepage](http://ipsec-tools.sourceforge.net/)

\[4\] [Internet Security Association and Key Management Protocol](https://en.wikipedia.org/wiki/Internet_Security_Association_and_Key_Management_Protocol)

\[5\] [Deprecating/removing racoon/ipsec-tools from Debian GNU/Linux and racoon from Debian/kfreebsd](https://lists.debian.org/debian-devel/2014/04/msg00075.html)

# Appendix

## Creating a Certificate Authority 

These are instructions for creating a certificate authority

Gernerate a private key:

```
openssl genpkey -algorithm RSA -out private_key.pem 4096
```

Generate a self signed certificate for the private key with a validity of 
approximately 10 years:

```
openssl req -new -x509 -days 3650 -key private_key.pem -out ca.crt
```
