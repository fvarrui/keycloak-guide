# Public Key Infraestructure

![](https://pki-tutorial.readthedocs.io/en/latest/_images/PKIProcess.png)

The idea with PKI is to have a trusted entity (CA) that will do the job of certifying that a given public key really belongs to a given person. This person must be identified by his name, email, address and other useful information that may allow to know who this person is. Once this work is done, the PKI emits a public certificate for this person. This certificate contains between others:

- All the information needed to identify this person (name, birth date,...).
- The public key of this person.
- The date of creation of the certificate.
- The date of revocation of the certificate.
- The digital signature of all this previous information issued by CA.

So now, if you want to send a private message to someone, you can ask for his certificate. When you received the certificate, must check the signature of the PKI who emitted it (the CA) and for the date of revocation. If verifications pass, then you can safely use the public key of the certificate to communicate with that person.

A certificate may be revocated before the date of end of validity has been reached. So a kind of list of revocated certificated (**CRL, Certificate Revocation List**) has to be maintained and accessed every time you want to use a certificate.

So, you need three types of certificates to configure KC:

- **Self-signed CA certificate**
  
  The **Certification Authority** (CA) is in charge of issuing the necessary certificates, using its own private key to sign auth server and users certificates. Its public key is also used for validating client certificates during X509 certificate authentication.
  
  To manage certificate-related issues you can use the `openssl` command. This command is used to create and manage certificates and certificate authority for your server. We are going to use this command to create a certificate and a self-signed CA. The self-signed CA is the highest level in the CA hierarchy, so it will be a **root-CA**.

- **Auth server certificate**
  
  The auth server (KC) needs this certificate to enable **mutual TLS** (mTLS), which is a requirement to use X.509 certificate authentication. Its public key has to be copied into the clients in order to trust the auth server. This certificate has to be signed by the CA.

- **User certificates**
  
  Each user also needs its own certificate (signed keypair), signed using the CA private key (issuer). This certificate must include user data in the subject (Subject DN): e.g. UID, NAME, SURNAME, E-MAIL, etc.

## Build a simple PKI

To construct the PKI, we first create the Root CA certificate (self-signed certificate). Then, we can issue certificates signed by the CA for servers and users.

### Create directory structure

To keep our PKI files well organized, we will create next directory structure:

```bash
mkdir conf ca crl certs
```

The `conf` directory holds configuration files, he `ca` directory holds CA resources, the `crl` directory holds CRLs (Certificate Revocation Lists), and the `certs` directory holds user certificates.

### Create Root CA

#### CA configuration file

We are going to use configuration files (`.cnf`) to make the process easier. So, first we create the `root-ca.cnf` file with next content:

```ini
# SoftQuadrat Root CA

# The [default] section contains global constants that can be referred to from
# the entire configuration file. It may also hold settings pertaining to more
# than one openssl command.

[ default ]
ca                      = root-ca               # CA name
dir                     = .                     # Top dir

# The next part of the configuration file is used by the openssl req command.
# It defines the CA's key pair, its DN, and the desired extensions for the CA
# certificate.

[ req ]
default_bits            = 2048                  # RSA key size
encrypt_key             = yes                   # Protect private key
default_md              = sha256                # MD to use
utf8                    = yes                   # Input is UTF-8
string_mask             = utf8only              # Emit UTF-8 strings
prompt                  = no                    # Don't prompt for DN
distinguished_name      = ca_dn                 # DN section
req_extensions          = ca_reqext             # Desired extensions

[ ca_dn ]
domainComponent         = "softquadrat.de"
organizationName        = "SoftQuadrat GmbH"
organizationalUnitName  = "Root CA"
commonName              = "Root CA"

[ ca_reqext ]
keyUsage                = critical,keyCertSign,cRLSign
basicConstraints        = critical,CA:true
subjectKeyIdentifier    = hash

# The remainder of the configuration file is used by the openssl ca command.
# The CA section defines the locations of CA assets, as well as the policies
# applying to the CA.

[ ca ]
default_ca              = root_ca               # The default CA section

[ root_ca ]
certificate             = $dir/ca/$ca.crt       # The CA cert
private_key             = $dir/ca/$ca/private/$ca.key # CA private key
new_certs_dir           = $dir/ca/$ca           # Certificate archive
serial                  = $dir/ca/$ca/db/$ca.crt.srl # Serial number file
crlnumber               = $dir/ca/$ca/db/$ca.crl.srl # CRL number file
database                = $dir/ca/$ca/db/$ca.db # Index file
unique_subject          = no                    # Require unique subject
default_days            = 3650                  # How long to certify for
default_md              = sha256                # MD to use
policy                  = match_pol             # Default naming policy
email_in_dn             = no                    # Add email to cert DN
preserve                = no                    # Keep passed DN ordering
name_opt                = ca_default            # Subject DN display options
cert_opt                = ca_default            # Certificate display options
copy_extensions         = none                  # Copy extensions from CSR
x509_extensions         = root_ca_ext           # Default cert extensions
default_crl_days        = 365                   # How long before next CRL
crl_extensions          = crl_ext               # CRL extensions

# Naming policies control which parts of a DN end up in the certificate and
# under what circumstances certification should be denied.

[ match_pol ]
domainComponent         = match                 # Must match 'softquadrat.de'
organizationName        = match                 # Must match 'SoftQuadrat GmbH'
organizationalUnitName  = optional              # Included if present
commonName              = supplied              # Must be present

[ user_pol ]
domainComponent         = match                 # Must match 'softquadrat.de'
organizationName        = match                 # Must match 'SoftQuadrat GmbH'
commonName              = supplied              # Must be present
organizationalUnitName  = optional              # Included if present
countryName             = optional              # Included if present
stateOrProvinceName     = optional              # Included if present
localityName            = optional              # Included if present
emailAddress            = optional              # Included if present
UID                     = supplied              # Must be present

# Certificate extensions define what types of certificates the CA is able to
# create.

[ root_ca_ext ]
keyUsage                = critical,keyCertSign,cRLSign
basicConstraints        = critical,CA:true
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always

[ user_ext ]
keyUsage                = critical,digitalSignature,keyEncipherment
basicConstraints        = CA:false
extendedKeyUsage        = clientAuth
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always

[ server_ext ]
keyUsage                = critical,digitalSignature,keyEncipherment
basicConstraints        = CA:false
extendedKeyUsage        = serverAuth,clientAuth
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always

# CRL extensions exist solely to point to the CA certificate that has issued
# the CRL.

[ crl_ext ]
authorityKeyIdentifier  = keyid:always
```

> To keep our PKI files well organized, we will create all `.cnf` files in `conf` directory.

#### Create CA directory structure

Create directories to store the Root CA related files:

```bash
mkdir -p ca/root-ca/private ca/root-ca/db
```

#### Create CA database

The database files must exist before the `openssl ca` command can be used. 

Next commands will create those files:

```bash
touch ca/root-ca/db/root-ca.db
touch ca/root-ca/db/root-ca.db.attr
echo 01 > ca/root-ca/db/root-ca.crt.srl
echo 01 > ca/root-ca/db/root-ca.crl.srl
```

#### Create CA certificate request (CSR)

With the next `openssl req -new` command we create a private key and a certificate signing request (CSR) for the root CA. You will be asked for a passphrase to protect the private key:

```bash
openssl req -new -config conf/root-ca.cnf -out ca/root-ca.csr -keyout ca/root-ca/private/root-ca.key
```

> The `openssl req` command takes its configuration from the `[req]` section of the configuration file.

Now we have created a certificate request in `ca/root-ca.csr` and its private key in `ca/root-ca/private/root-ca.key`.

#### Create CA certificate

With the next `openssl ca` command we issue a root CA certificate based on the CSR. The root certificate is self-signed and serves as the starting point for all trust relationships in the PKI:

```bash
openssl ca -selfsign -config conf/root-ca.cnf -in ca/root-ca.csr -out ca/root-ca.crt -extensions root_ca_ext -batch
```

> The `openssl ca` command takes its configuration from the `[ca]` section of the configuration file.

Now we have the self-signed CA certificate in `ca/root-ca.crt` (signed with its own private key).

#### Create CA PKCS#12 bundle (optional)

```bash
openssl pkcs12 -export -name "Root CA" -inkey ca/root-ca/private/root-ca.key -in ca/root-ca.crt -out ca/root-ca.p12
```

Now we have created a PKCS#12 bundle in `ca/root-ca.p12`, containing our self-signed CA certificate and its private key (it should be protected with a passphrase).

#### Import CA PKCS#12 certificate bundle into a Java Keystore (optional)

We can also change the format in which our certificate and its private key are stored. 

Next command will create a JKS from our previous P12 file:

```bash
keytool -importkeystore -srckeystore ca/root-ca.p12 -srcstoretype pkcs12 -destkeystore ca/root-ca.jks -deststoretype JKS
```

> In this case we need to have Java installed because `keytool`is available in JDK.

After running last command, we have created the `ca/root-ca.jks` keystore file containing the CA certificate and the private key (it also should be protected with a passphrase).

### Create user certificates

First we have to create the request configuration file `conf/user.cnf` with the next content:

```ini
# User certificate request

# This file is used by the openssl req command. Since we cannot know the DN in
# advance the user is prompted for DN information.

[ req ]
default_bits            = 2048                  # RSA key size
encrypt_key             = yes                   # Protect private key
default_md              = sha256                # MD to use
utf8                    = yes                   # Input is UTF-8
string_mask             = utf8only              # Emit UTF-8 strings
prompt                  = yes                   # Prompt for DN
distinguished_name      = user_dn               # DN template
req_extensions          = user_reqext           # Desired extensions

[ user_dn ]
domainComponent         = "Domain Component (eg, acme.com)"
organizationName        = "Organization Name (eg, ACME)"
organizationalUnitName  = "Organization Unit Name (eg, IT department)"
commonName              = "Common Name (eg, full name)"
emailAddress            = "Email Address (eg, name@fqdn)"
UID                     = "User Identifier (eg, username)"

domainComponent_default         = "softquadrat.de"
organizationName_default        = "SoftQuadrat GmbH"
organizationalUnitName_default  = "Datasqill"


[ user_reqext ]
keyUsage                = critical,digitalSignature,keyEncipherment
extendedKeyUsage        = clientAuth
subjectKeyIdentifier    = hash
```

Then, with the `openssl req -new` command we create the private key and CSR (certificate request) for an user certificate. We use the previous request configuration file specifically prepared for the task. When prompted enter the required DN components (leave fields empty if are not necessary):

```bash
openssl req -new -config conf/user.cnf -out certs/fran.csr -keyout certs/fran.key
```

Finally, we use the signing CA to sign the user certificate:

```bash
openssl ca -config conf/root-ca.cnf -in certs/fran.csr -out certs/fran.crt -extensions user_ext -policy user_pol -batch
```

>  The certificate type is defined by the extensions we attach (`user_ext` in this case).

A copy of the certificate is saved in the certificate archive under the name `ca/root-ca/XX.pem` (`XX` being the certificate serial number in hexadecimal).

### Create servers certificates

Create a request configuration file `conf/server.cnf` with the next content:

```ini
# Server certificate request

# This file is used by the openssl req command.

[ req ]
default_bits            = 2048                  # RSA key size
encrypt_key             = yes                   # Protect private key
default_md              = sha256                # MD to use
utf8                    = yes                   # Input is UTF-8
string_mask             = utf8only              # Emit UTF-8 strings
prompt                  = yes                   # Prompt for DN
distinguished_name      = server_dn             # DN template
req_extensions          = server_reqext         # Desired extensions

[ server_dn ]
domainComponent         = "Domain Component (eg, acme.com)"
organizationName        = "Organization Name (eg, ACME)"
organizationalUnitName  = "Organization Unit Name (eg, IT department)"
commonName              = "Common Name (eg, FQDN)"

domainComponent_default         = "softquadrat.de"
organizationName_default        = "SoftQuadrat GmbH"
organizationalUnitName_default  = "Datasqill"


[ server_reqext ]
keyUsage                = critical,digitalSignature,keyEncipherment
extendedKeyUsage        = serverAuth,clientAuth
subjectKeyIdentifier    = hash
subjectAlternativeName  = DNS:server.domain.name
```

> Don't forget to set the server domain name with the `DNS:` preffix. `subjectAlternativeName` can be a comma separated list of domain names. This field is important so that certificate can not be used for different servers than specified in the SAN field.

As we did to create user certificates, with the `openssl req -new` command we create the private key and CSR (certificate request) for ther server certificate. When prompted enter the required DN components (leave fields empty if are not necessary):

```bash
openssl req -new -config conf/server.cnf -out certs/my-server.csr -keyout certs/my-server.key
```

Sign the request using the signing CA private key:

```bash
openssl ca -config conf/root-ca.cnf -in certs/my-server.csr -out certs/my-server.crt -extensions server_ext -policy user_pol -batch
```

> Note that in this case we are using the same naming policy as for user certificates (`user_pol`), but a different extension (`server_ext`), since we want to set the same DN components as for user certificates.

# References

- [OpenSSL | Cryptography and SSL/TLS Toolkit](https://www.openssl.org/)
- [OpenSSL PKI Tutorial](https://pki-tutorial.readthedocs.io/en/latest/)
- [Simple PKI | OpenSSL PKI Tutorial](https://pki-tutorial.readthedocs.io/en/latest/simple/index.html)
