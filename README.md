# OpenSSL
A simple guide to OpenSSL.
OpenSSL allows you to create SSL certificates within your network without a certfied Certificate Authority.
I used a tutorial made by theurbanpenguin to make this guide.This guide is bassically a summary of his tutorial.If anything is unclear here please check out his amazing video.https://www.youtube.com/watch?v=d8OpUcHzTeg&t=22s

## Basic Introduction
Certificates are used for encrypted communications between devices.SSL are commonly used for encrypting communications on the web but it can also be used with LAN.Many organizations would use a self signed Certifcate Authority to reduce overhead and increase efficency over using a certified third-party Certificate Authority.In this tutorial i will be creating a root Certificate Authority , an intermediate certificate Authority , and Servers.The root certificate authority is the central authority.The intermediate 
handles certificate requests on behalf of the root Certificate Authority.

## Create directories
Create the structure for your Certificates Authorities. In this tutorial we will be doing everything on the same system.Typically you will have the root certificate in a seperate offline system for sercurity reasons 

```
mkdir -p ca/{root_ca,sub_ca,server}/{private,certs,newcerts,crl,csr}

```
We just created three directories.The root_ca , sub_ca, and server . Within each directory we created sub-directories private,certs,newcerts,crl,csr

Each of the private directory needs to be secret and shouldnt be access by others so we set the permissions of it to RWX for owner only 

```
chmod -v 700 ca/{root_ca,sub_ca,server}/private

```
## Create index and serial file 
These two files are only used by the CA.The index file contains information about the certificate that it has handed out and other metadata. 
The serial numbers that are stored in the serial file is an indentifer of the certificate it has handed out and is stored in the serial file.

Generate serial number by 
```
openssl rand -hex 16 > ca/root_ca/serial
openssl rand -hex 16 > ca/sub_ca/serial

```
## Create private key for both CA and Server 
The encryption method will be different for the CA and the server.We would use a "less secure" encryption method and sercurity measure for the server to create less overhead.The private key for will also be guarded with a passphrase for the CA but not the server.
```
#for root CA
openssl genrsa -aes256 -out root-ca/private/root_key 4096
#for sub CA
openssl genrsa -aes256 -out sub-ca/private/sub_key 4096
#for server
openssl genrsa -out server/private/server_key 2048
```
## Create the certificates 
Create a configuration file using this. PS( Again this guide is bassically a copy paste of the tutorial from theurbanpenguin.Go check out his video if anything is confusing)
```
[ca]
#/root/ca/root-ca/root-ca.conf
#see man ca
default_ca    = CA_default

[CA_default]
dir     = /root/ca/root-ca
certs     =  $dir/certs
crl_dir    = $dir/crl
new_certs_dir   = $dir/newcerts
database   = $dir/index
serial    = $dir/serial
RANDFILE   = $dir/private/.rand

private_key   = $dir/private/ca.key
certificate   = $dir/certs/ca.crt

crlnumber   = $dir/crlnumber
crl    =  $dir/crl/ca.crl
crl_extensions   = crl_ext
default_crl_days    = 30

default_md   = sha256

name_opt   = ca_default
cert_opt   = ca_default
default_days   = 365
preserve   = no
policy    = policy_strict

[ policy_strict ]
countryName   = supplied
stateOrProvinceName  =  supplied
organizationName  = match
organizationalUnitName  =  optional
commonName   =  supplied
emailAddress   =  optional

[ policy_loose ]
countryName   = optional
stateOrProvinceName  = optional
localityName   = optional
organizationName  = optional
organizationalUnitName   = optional
commonName   = supplied
emailAddress   = optional

[ req ]
# Options for the req tool, man req.
default_bits   = 2048
distinguished_name  = req_distinguished_name
string_mask   = utf8only
default_md   =  sha256
# Extension to add when the -x509 option is used.
x509_extensions   = v3_ca

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address
countryName_default  = GB
stateOrProvinceName_default = England
0.organizationName_default = TheUrbanPenguin Ltd

[ v3_ca ]
# Extensions to apply when createing root ca
# Extensions for a typical CA, man x509v3_config
subjectKeyIdentifier  = hash
authorityKeyIdentifier  = keyid:always,issuer
basicConstraints  = critical, CA:true
keyUsage   =  critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions to apply when creating intermediate or sub-ca
# Extensions for a typical intermediate CA, same man as above
subjectKeyIdentifier  = hash
authorityKeyIdentifier  = keyid:always,issuer
#pathlen:0 ensures no more sub-ca can be created below an intermediate
basicConstraints  = critical, CA:true, pathlen:0
keyUsage   = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
# Extensions for server certificates
basicConstraints  = CA:FALSE
nsCertType   = server
nsComment   =  "OpenSSL Generated Server Certificate"
subjectKeyIdentifier  = hash
authorityKeyIdentifier  = keyid,issuer:always
keyUsage   =  critical, digitalSignature, keyEncipherment
extendedKeyUsage  = serverAuth
```
Using the configuration file we will create a Certifcate from the private key we generated before for the root CA
 
```
#first navavigate into the root ca directory
openssl req -config root-ca.conf -key private/ca.key -new -x509 -days 7500 -sha256 -extensions v3_ca -out certs/ca.crt 
#after entering password we will need to configure the information of the certificate.The common name is usually the FQDN of the host

```
-config points to the config file we just created , -key is the private key we just generated , to specify we want to create a new certificate we do -new -x509 , the -days represent how long do you want the certificate to be valid for , we want to digest this in sha256 , -extensions is for what we can use later , and -out is specifying where we want to store the Certificate.

To see the certifcate we just created for the root CA
 
``` 
openssl x509 -noout -in certs/ca.crt -text 

``` 
For the intermediate CA we would create a request for a certificate from the root CA instead of creating one.The root CA would then sign the certificate request and we then we can use the signed certificate.

```
#navigate to the intermediate CA directory 
#create a configuration file just like you did for the root CA
#create a sign request to the root CA
open ssl req -config sub_ca.conf -new -key private/sub_ca.key -sha256 -out csr/sub_ca.csr
``` 
Now we want to use the root CA to authorize this 
We store the certificate request on the directory of the sub CA so we will specify where the request is in the command and we also want to specify where to output the signed request. 

```
#navigate to root CA
openssl ca -config root_ca.conf -extensions v3_intermediate_ca -days 3650 -notext -in ../sub_ca/csr/sub_ca.csr -out ../sub_ca/certs/sub_ca.crt

```
## Create Server Certificate 
Create a request for a certificate 
```
#navigate to server directory 
openssl req -key private/server.key -new -sha256 -out csr/server.csr 
#It will ask for configuration , the most important one is the common name or FQDN.This what the client will use to try and connect to the server 

```
Sign the request using the intermediate CA

``` 
#Go to the sub CA directory 
openssl ca -config sub-ca.conf -extensions server_cert -days 365 -notext -in ../server/csr/server.csr -out ../server/certs/server.crt

```

## Conclusions

AGAIN if anything here is confusing , go check out the video made by theurbanpenguin.https://www.youtube.com/watch?v=d8OpUcHzTeg&t=22s.
















