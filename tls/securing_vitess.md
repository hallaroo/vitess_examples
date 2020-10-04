# Securing Vitess using TLS

## Introduction

Vitess has a number of different components that, in most real-world
configurations, connect to each other over the network. Many organizations
require, for compliance or practical reasons, that these communications
be encrypted and/or authenticated. This guide will provide an overview
of these client/server combinations between components, what the
encryption and authentication options are, and a walkthrough on how
to configure Vitess on how to use them.

TODO: link in diagram

We should note that Vitess has two mostly distinct purposes 
for components to talk to each other:

  * Data path - e.g.:
    * Main query path:
      * application -> vtgate
      * vtgate -> vttablet
      * vttablet -> MySQL
    * vttablet -> vtablet:  vreplication within or across shards
    * MySQL -> MySQL:  within-shard replication
  * Control or meta-data path - e.g.:
    * vtgate -> topology server
    * vttablet -> topology server
    * vtctld -> topology server
    * vtctlclient -> vtctld
    * vtctld -> vttablet
    * administrator using web browser -> vtgate web UI
    * administrator using web browser -> vttablet web UI
    * administrator using web browser -> vtctld web UI
    * Metrics scraper (e.g. Prometheus) -> vtgate web port
    * Metrics scraper (e.g. Prometheus) -> vttablet web port
    * Metrics scraper (e.g. Prometheus) -> vtctld web port

Note that the sensitive information mainly flows over the data path,
and depending on your deployment model, you may not have to encrypt
the control or meta-data path.  We recommend that you evaluate your
needs in the context of your compliance directives, threat model and
risk management framework.

It should be noted that while Vitess provides the mechanism for
securing these communication channels, it does not manage the
important certificate management tasks like:
  * Securely generating private keys
  * Issuing server certificates
  * Issuing, if necessary, client certificates
  * Certificate rotation
  * Certificate audit

Indeed, the hardest part of deploying TLS with Vitess in a large
organization may be to integrate with whatever certificate policies
and procedures the organization mandates. It should be noted that
the manual issuing and rotation of certificates in a Vitess environment
of a non-trivial size is impractical, and some provisioning
and configuration management automation will need to be built.

## Protocols involved

Of all the data, meta-data and control paths enumerated above, they use one
of three protocols:
  * MySQL binary protocol
  * gRPC (using HTTP/2 as transport)
  * HTTP

## Encryption

All three the protocol types above use TLS in one form or another to
encrypt communications.  Therefore the basics around encrypting a specific
client to server communication is straightforward:

  * Server-side:
    * Generate a CA private key and cert.
    * Generate a private key for the server component involved.
    * Generate a CSR using the private key.
    * Use the CA key material to use the CSR to generate a server cert
    * Install the server cert and private key using the appropriate Vitess
      options for the component in question.
    * If required, adjust other Vitess component options to enforce/require
      TLS-only communications.

## Server authentication

In addition to encrypting the connection, you may want or need to
configure client-side server authentication.  This is the process by which
the client verifies that the server it is trying to establish a TLS
connection to is who they claim to be, and not an imposter or
man-in-the-middle.  We achieve this by:

  * Client-side:
    * Install the CA cert used by your certificate issuing process to sign
      the server component certificates.
    * Adjust the Vitess client component options to verify the server
      certficate using the installed CA cert.  This would typically involve
      specifying the CA cert, as well as the server or common name to
      expect from the server component, if it isn't the same as the
      DNS name (or has an IP SAN configured).

## Client authentication

Client authentication in Vitess can take two forms, depending on the protocol
in question:
  * TLS client certificate authentication (also known as mTLS)
  * Username/password authentication;  this is only an option for the
    connections involving the MySQL protocol.


# Walkthroughs

With the preliminaries concluded, we can now move on to a walkthrough of
how to setup the various TLS component combinations.  We will start with the
data path.

## Certificate generation

As discussed above, large organizations will often have established tools
to secure a TLS certificate hierarchy and issue certificates. For the
purpose of these walkthroughs, we could use bare `openssl` commands to
step through every detail. However, since we consider this an implementation
detail that is likely to vary from user to user, we will leverage a
shell-script-based tool called [easy-rsa](https://github.com/OpenVPN/easy-rsa).
This tool has been around for many years as part of the OpenVPN project,
and can perform all the steps to establish a CA, generate server certificates
and also client certificates if desired. Since `easy-rsa` is just a set of
shell scripts, if you require a closer understanding of how every step works,
this is easy to discover as well.  Lastly, `easy-rsa` can be used in
production, and can easily manage thousands of certificates, if desired.

We will use the newest `easy-rsa` release at the time of writing, version
3.0.8.

## Installing easy-rsa

* Create a directory to install and run `easy-rsa` from, download and unpack
the tool:

  ```
  $ echo $HOME
  /home/user
  $ mkdir ~/CA
  $ cd ~/CA/
  $ wget https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.8/EasyRSA-3.0.8.tgz
  .
  .
  2020-10-02 16:10:22 (604 KB/s) - ‘EasyRSA-3.0.8.tgz’ saved [48907/48907]
  $ tar zxf EasyRSA-3.0.8.tgz 
  $ mv EasyRSA-3.0.8/easyrsa .
  $ mv EasyRSA-3.0.8/openssl-easyrsa.cnf .
  $ mv EasyRSA-3.0.8/x509-types .
  $ mv EasyRSA-3.0.8/vars.example vars
  $ rm -rf EasyRSA-3.0.8
  ```

* Edit the `vars` file appropriately for your setup.
  For the purposes of this walkthrough we will just append the following
  lines at the end of the file. Please adjust for your needs:

  ```
  set_var EASYRSA_DN             "org"
  set_var EASYRSA_REQ_COUNTRY    "US"
  set_var EASYRSA_REQ_PROVINCE   "California"
  set_var EASYRSA_REQ_CITY       "Mountain View"
  set_var EASYRSA_REQ_ORG        "PlanetScale Inc"
  set_var EASYRSA_REQ_EMAIL      "carequest@planetscale.com"
  set_var EASYRSA_REQ_OU         "Operations"
  set_var EASYRSA_KEY_SIZE       2048
  set_var EASYRSA_ALGO           rsa
  set_var EASYRSA_CA_EXPIRE      3650
  set_var EASYRSA_CERT_EXPIRE    1095
  ```

* Bootstrap your CA.  During the second step you will be prompted for
  a password. Do not forget this password! You will
  not be able to recover it.  For the answers after the password
  prompt, you should be able to just hit enter multiple times, you
  have already configured it in the `vars` file above.

  ```
  $ cd ~/CA/
  $ ./easyrsa init-pki

  Note: using Easy-RSA configuration from: /home/user/CA/vars

  init-pki complete; you may now create a CA or requests.
  Your newly created PKI dir is: /home/user/CA/pki

  $ ./easyrsa build-ca

  Note: using Easy-RSA configuration from: /home/user/CA/vars
  Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020

  Enter New CA Key Passphrase: 
  Re-Enter New CA Key Passphrase: 
  Generating RSA private key, 2048 bit long modulus (2 primes)
  ...............................................................................+++++
  e is 65537 (0x010001)
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [US]:
  State or Province Name (full name) [California]:
  Locality Name (eg, city) [Mountain View]:
  Organization Name (eg, company) [PlanetScale Inc]:
  Organizational Unit Name (eg, section) [Operations]:
  Common Name (eg: your user, host, or server name) [Easy-RSA CA]:
  Email Address [carequest@planetscale.com]:

  CA creation complete and you may now import and sign cert requests.
  Your new CA certificate file for publishing is at:
  /home/user/CA/pki/ca.crt
  ```

Your CA is now configured.  Generating certificates will be trivial now.


## Application to vtgate

While applications can connect to vtgate using gRPC, the vast majority of
Vitess users only use the MySQL protocol. When using the MySQL protocol, most
users will use username/password for client authentication, although it is
also possible to configure TLS client certificate authentication. We will
assume the use of username/password authentication:

* For each vtgate you should generate a server private key and certificate.
  We will do this in two steps:  First we generate a private key and
  certificate request.  We will then use the CA to sign that request to
  produce the server certificate.
  For the the prompts during `gen-req`, you can just hit enter.
  You will be prompted to type `yes` and enter the CA password during the
  `sign-req` phase.

  ```
  $ cd ~/CA/
  $ ./easyrsa gen-req vtgate1 nopass

  Note: using Easy-RSA configuration from: /home/user/CA/vars
  Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020
  Generating a RSA private key
  ............................+++++
  writing new private key to '/home/user/CA/pki/easy-rsa-178308.W6uc3G/tmp.Iqlvgf'
  -----
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [US]:
  State or Province Name (full name) [California]:
  Locality Name (eg, city) [Mountain View]:
  Organization Name (eg, company) [PlanetScale Inc]:
  Organizational Unit Name (eg, section) [Operations]:
  Common Name (eg: your user, host, or server name) [vtgate1]:
  Email Address [carequest@planetscale.com]:

  Keypair and certificate request completed. Your files are:
  req: /home/user/CA/pki/reqs/vtgate1.req
  key: /home/user/CA/pki/private/vtgate1.key

  $ ./easyrsa sign-req server vtgate1

  Note: using Easy-RSA configuration from: /home/user/CA/vars
  Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020


  You are about to sign the following certificate.
  Please check over the details shown below for accuracy. Note that this request
  has not been cryptographically verified. Please be sure it came from a trusted
  source or that you have verified the request checksum with the sender.

  Request subject, to be signed as a server certificate for 1095 days:

  subject=
      countryName               = US
      stateOrProvinceName       = California
      localityName              = Mountain View
      organizationName          = PlanetScale Inc
      organizationalUnitName    = Operations
      commonName                = vtgate1
      emailAddress              = carequest@planetscale.com


  Type the word 'yes' to continue, or any other input to abort.
    Confirm request details: yes
  Using configuration from /home/user/CA/pki/easy-rsa-177552.IsttQK/tmp.NA5kv0
  Enter pass phrase for /home/user/CA/pki/private/ca.key:
  Check that the request matches the signature
  Signature ok
  The Subject's Distinguished Name is as follows
  countryName           :PRINTABLE:'US'
  stateOrProvinceName   :ASN.1 12:'California'
  localityName          :ASN.1 12:'Mountain View'
  organizationName      :ASN.1 12:'PlanetScale Inc'
  organizationalUnitName:ASN.1 12:'Operations'
  commonName            :ASN.1 12:'vtgate1'
  emailAddress          :IA5STRING:'carequest@planetscale.com'
  Certificate is to be certified until Oct  4 00:07:58 2023 GMT (1095 days)

  Write out database with 1 new entries
  Data Base Updated

  Certificate created at: /home/user/CA/pki/issued/vtgate1.crt
  ```

* Our certificate has now been issued, and we can use the private key file
  in `/home/user/CA/pki/private/vtgate1.key` along with the issued
  server certificate in `/home/user/CA/pki/issued/vtgate1.crt` to 
  configure vtgate for using TLS with MySQL clients.  First we copy the
  private key and server certificate to the appropriate configuration
  directory, and then tighten up the file permissions and owernership.
  This will differ in your environment:

  ```
  $ mkdir ~/config/
  $ cp /home/user/CA/pki/private/vtgate1.key ~/config/
  $ cp /home/user/CA/pki/issued/vtgate1.crt ~/config/
  $ chown vtgate:vtgate ~/config/vtgate1.*
  $ chmod 400 ~/config/vtgate1.*
  ```

* Now, we can add the options to vtgate to use the above private key
  and server certificate.  Modify the vtgate commandline or startup
  script to add the following parameters:

  ```
  -mysql_server_ssl_key ~/config/vtgate1.key -mysql_server_ssl_cert ~/config/vtgate1.crt -mysql_server_require_secure_transport
  ```

  You can now start/restart the vtgate instance.  Any vtgate connections
  from now on will be required to use TLS, so you may have to reconfigure
  your applications/clients. In addition, to avoid man-in-the-middle
  attacks, you may want the clients to verify the server by validating
  the server certificate against the CA cert.  Here is an example using the
  MySQL CLI client.  Exact options will vary between MySQL versions, this is
  using a recent (8.0.21) MySQL client:

  ```
  $ cp /home/user/CA/pki/ca.crt /var/tmp/ca.crt
  $ mysql -u mysql_user -p -h 127.0.0.1 -P 15306 --ssl-mode=VERIFY_CA --ssl-ca=/var/tmp/ca.crt
  Enter password: 
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 3
  Server version: 5.7.9-Vitess MySQL Community Server - GPL
  .
  .
  .
  mysql> \s
  --------------
  mysql  Ver 8.0.20-11 for Linux on x86_64 (Percona Server (GPL), Release 11, Revision 159f0eb)

  Connection id:          3
  Current database:
  Current user:           vt_app@localhost
  SSL:                    Cipher in use is ECDHE-RSA-AES128-GCM-SHA256
  Current pager:          stdout
  Using outfile:          ''
  Using delimiter:        ;
  Server version:         5.7.9-Vitess MySQL Community Server - GPL
  Protocol version:       10
  Connection:             127.0.0.1 via TCP/IP
  .
  .
  ```

  The above MySQL CLI output shows that the connection is encrypted, and that
  the server (vtgate) was successfully validated using the CA certificate.
  If the server certificate could not be validated using the CA certificate,
  an error similar to this would have been seen:

  ```
  ERROR 2026 (HY000): SSL connection error: error:14090086:SSL routines:SSL3_GET_SERVER_CERTIFICATE:certificate verify failed
  ```

  If TLS was not setup on the vtgate at all, an error like this could have 
  resulted:

  ```
  ERROR 2026 (HY000): SSL connection error: SSL is required but the server doesn't support it
  ```



## vttablet to MySQL

vttablet communications to MySQL are often via local unix socket or via
a TCP connection on localhost. In a case like this, it is probably unnecessary
to configure encryption between vttablet and MySQL, since the traffic never
leaves the local machine/VM. However, in some deployment models vttablet and
MySQL are running on different hosts, and you may want vttablet to use TLS to
speak to MySQL.  We will not cover configuring MySQL to use TLS certificates
extensively here, just the minimum.  Please consult the MySQL documentation
for further information.  Again, we will also assume that vttablet will be
using MySQL username/password client authentication.

* Generate a server certificate for our MySQL instance using our CA:

  ```
  $ cd ~/CA/
  $ ./easyrsa gen-req mysql1 nopass
  .
  .
  .
  Keypair and certificate request completed. Your files are:
  req: /home/user/CA/pki/reqs/mysql1.req
  key: /home/user/CA/pki/private/mysql1.key

  $ ./easyrsa sign-req server mysql1
  .
  .
  .
  Write out database with 1 new entries
  Data Base Updated

  Certificate created at: /home/user/CA/pki/issued/mysql1.crt
  ```

* Copy the files `/home/user/CA/pki/private/mysql1.key` and
  `/home/user/CA/pki/issued/mysql1.crt` to the MySQL server in
  the appropriate locations, securing their ownership and 
  permissions appropriately.

* Configure the MySQL server options `ssl-key` and `ssl-cert`
  appropriately to point to where you placed the private key
  and certificate above. 
  Note that these options do not require clients to use TLS,
  but is optional.  If you need to require all TCP/IP clients to use
  TLS, you can use the MySQL server option `require_secure_transport`,
  or you can enforce it on a per MySQL user basis by using
  the `REQUIRE SSL` option when creating or altering a MySQL-level
  user.  See the MySQL documentation for details.
  Restart your MySQL server to make these MySQL server option
  configuration changes active.

* Now, configure vttablet to connect to MySQL using the necessary
  parameters, verifying the CA certificate:

  ```
  $ cp /home/user/CA/pki/ca.crt ~/config/
  ```

  Add the vttablet parameters:

  ```
  -db_ssl_ca /home/user/config/ca.crt -db_flags 1073743872 -db_server_name mysql1
  ```
  
  and restart vttablet.  Note that the `db_server_name` parameter value will
  differ depending on your issued certificate common name;  and is unnecessary
  if the certificate common name matches the DNS name vttablet is using
  to connect to the MySQL server.

  If you just wish to encrypt the vttablet -> MySQL server communication
  and you do not care about server certificate validation, you can just use the
  vttablet flags:

  ```
  -db_flags 2048
  ```
  
  instead.

## vtgate to vttablet

In Vitess, communication between vtgate and vttablet instances are via gRPC.
gRPC uses HTTP/2 as a transport protocol, but by default this is not encrypted
in Vitess.  To secure this data path you need to, at a minimum, configure
TLS for gRPC on the server (vttablet) side. You may also want to verify
the server TLS certificate from the client (vtgate) side using the server CA
certificate.


* First, generate a certificate for use by vttablet:

  ```
  $ cd ~/CA/
  $ ./easyrsa gen-req vttablet1 nopass

  Note: using Easy-RSA configuration from: /home/user/CA/vars
  Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020
  Generating a RSA private key
  ..................................+++++
  .....+++++
  writing new private key to '/home/user/CA/pki/easy-rsa-209692.tdDNNt/tmp.hwhw8x'
  -----
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [US]:
  State or Province Name (full name) [California]:
  Locality Name (eg, city) [Mountain View]:
  Organization Name (eg, company) [PlanetScale Inc]:
  Organizational Unit Name (eg, section) [Operations]:
  Common Name (eg: your user, host, or server name) [vttablet1]:
  Email Address [carequest@planetscale.com]:

  Keypair and certificate request completed. Your files are:
  req: /home/user/CA/pki/reqs/vttablet1.req
  key: /home/user/CA/pki/private/vttablet1.key

  $ ./easyrsa sign-req server vttablet1

  Note: using Easy-RSA configuration from: /home/user/CA/vars
  Using SSL: openssl OpenSSL 1.1.1g FIPS  21 Apr 2020


  You are about to sign the following certificate.
  Please check over the details shown below for accuracy. Note that this request
  has not been cryptographically verified. Please be sure it came from a trusted
  source or that you have verified the request checksum with the sender.

  Request subject, to be signed as a server certificate for 1095 days:

  subject=
      countryName               = US
      stateOrProvinceName       = California
      localityName              = Mountain View
      organizationName          = PlanetScale Inc
      organizationalUnitName    = Operations
      commonName                = vttablet1
      emailAddress              = carequest@planetscale.com


  Type the word 'yes' to continue, or any other input to abort.
    Confirm request details: yes
  Using configuration from /home/user/CA/pki/easy-rsa-209844.f9wDrk/tmp.3rww6R
  Enter pass phrase for /home/user/CA/pki/private/ca.key:
  Check that the request matches the signature
  Signature ok
  The Subject's Distinguished Name is as follows
  countryName           :PRINTABLE:'US'
  stateOrProvinceName   :ASN.1 12:'California'
  localityName          :ASN.1 12:'Mountain View'
  organizationName      :ASN.1 12:'PlanetScale Inc'
  organizationalUnitName:ASN.1 12:'Operations'
  commonName            :ASN.1 12:'vttablet1'
  emailAddress          :IA5STRING:'carequest@planetscale.com'
  Certificate is to be certified until Oct  4 20:23:48 2023 GMT (1095 days)

  Write out database with 1 new entries
  Data Base Updated

  Certificate created at: /home/user/CA/pki/issued/vttablet1.crt

  $ cp /home/user/CA/pki/private/vttablet1.key ~/config/
  $ cp /home/user/CA/pki/issued/vttablet1.crt ~/config/
  $ chmod 400 ~/config/vttablet1.*
  ```

* To configure vttablet to use a server certificate for its gRPC server, add:

  ```
  -grpc_cert /home/user/config/vttablet1.crt -grpc_key /home/user/config/vttablet1.key 
  ```

  to the vttablet parameters.  Note that adding these options *enforces* TLS
  only gRPC connections to this vttablet instace from that point onwards.

* This means that you will need to add the following option to your vtgate
  instances to successfully connect to this vttablet instance from this point
  forward:

  ```
  -tablet_grpc_server_name vttablet1 -tablet_grpc_ca /home/user/config/ca.crt 
  ```

* Adding this option to a vtgate instance will require all vttablet instances
  this vtgate connects to to be configured for TLS as well. This is
  unfortunately an all-or-nothing proposition, there is no incremental
  migration to using TLS in this case.

  If you have vtgate instances accessing your vttablet instance after you
  have configured TLS on the vttablet side, you may see errors like this
  in the vttablet logs:

  ```
  W1004 13:34:16.352458  212354 server.go:650] grpc: Server.Serve failed to complete security handshake from "[::1]:51492": tls: first record does not look like a TLS handshake
  ```

  Conversely, if you have configured the TLS parameters on the vtgate side
  and the vtgate instance is still trying to connect to vttablet instances
  that are not configured with the correct TLS options, you might see
  errors like this in the vtgate logs:

  ```
  W1004 14:38:29.383672  214179 tablet_health_check.go:323] tablet cell:"zone1" uid:101  healthcheck stream error: Code: UNAVAILABLE
  vttablet: rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: connection error: desc = "transport: authentication handshake failed: tls: first record does not look like a TLS handshake"
  ```