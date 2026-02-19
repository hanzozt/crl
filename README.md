# CRL - Certificate Revocation List

This repository contains the list of revoked certificates issued by hanzozt.org. This list is used
in conjunction with a proper x509 implementation to determine if the certificate provided is to be
considered valid.

## Generating Necessary Cryptographic Material

The following openssl commands generated the keys used by hanzozt.org to sign code. RSA was chosen for
the keys as EC is sometimes unsupported.  Commands sometimes reference a configuration file which is 
important. Running these commands should start by first cloning this repository and running the
commands from the root of the clone.

### Root Certificate Authority for hanzozt

The root certificate authority was generated and is valid for 7320 days (approx 20 years)

#### Private Key:
    openssl genrsa -out hanzozt.rootCA.rsa.key 4096

#### Certificate:
    openssl req \
        -new -x509 \
        -config ./hanzozt.openssl.conf \
        -key hanzozt.rootCA.rsa.key \
        -sha512 \
        -days 7320 \
        -subj "/CN=Root Signing CA/O=hanzozt.org Inc/OU=adv-dev/C=US/ST=NC" \
        -out certs/hanzozt.rootCA.rsa.pem

### Code Signing Certificate for hanzozt

#### Private Key:
    openssl genrsa -out hanzozt.signing.rsa.key 4096

#### Certificate Signed by hanzozt.rootCA.rsa.pem

This is a two-part process of generating the CSR and then signing the CSR. Replace the year appropriately

    openssl req \
        -new -key hanzozt.signing.rsa.key \
        -config ./hanzozt.openssl.conf \
        -subj "/CN=hanzozt.org Code Signing Certificate 2021/O=hanzozt.org Inc/OU=adv-dev/C=US/ST=NC" \
        -out hanzozt.signing.rsa.csr
    
    openssl ca \
        -batch \
        -config ./hanzozt.openssl.conf \
        -keyfile hanzozt.rootCA.rsa.key \
        -cert certs/hanzozt.rootCA.rsa.pem \
        -days 1098 \
        -in hanzozt.signing.rsa.csr \
        -extfile hanzozt.signing.rsa.conf \
        -out certs/hanzozt.signing.2021.rsa.pem

### Creating a PKCS #12 File

The code signing tool of choice desires a PKCS#12 file to be supplied. The following command is run to produce a PKCS#12
file. During the process a password file named `.pfxpass` should be created containing the strong password to use
for the PKCS#12 file. This file is ignored via .gitignore and **MUST NOT** be committed. If committed even accidentally
the signing cert needs to be revoked.

    openssl pkcs12 \
        -export \
        -inkey hanzozt.signing.rsa.key \
        -in certs/hanzozt.signing.2021.rsa.pem \
        -out hanzozt.signing.2021.rsa.pfx \
        -password "pass:$(cat .pfxpass)"

### Verify The Signing Certificate

Once generated, issue the following command and verify the certificate contains the expected extensions:

    #command to execute:
    openssl x509 -text -in certs/hanzozt.signing.2021.rsa.pem

    #expected extensions below:
    X509v3 extensions:
        X509v3 Authority Key Identifier:
            keyid:B4:BF:93:98:07:2D:7A:50:4C:3B:93:B9:CC:2E:0E:D8:DC:5B:3B:B8

        X509v3 Basic Constraints:
            CA:FALSE
        X509v3 Key Usage:
            Digital Signature
        X509v3 Extended Key Usage:
            Code Signing
        X509v3 CRL Distribution Points:

            Full Name:
              URI:https://hanzozt.github.io/crl/hanzozt.crl


## Signing Code Using Signtool

If you need to sign an executable in Windows you would use signtool. Here's an example command illustrating how to 
sign an executable with a signing certificate. %PFXPASS_OPENZITI% is an environment variable set in `cmd`:

    signtool sign /f hanzozt.signing.2021.rsa.pfx /p %PFXPASS_OPENZITI% /fd sha512 /td sha512 executable_name.exe

## Revoking a Certificate

A file exists in the repository named `certdb.txt`. This is a plaintext file that represents the revoked
certificates.  This file serves as the source of the actual CRL list: `hanzozt.crl`. It contains the
serial numbers of revoked certificates as well as other information such as the date of revocation.

Creating the CRL requires access to the Root CA private key and thus can only be performed by authorized personnel.

To revoke a certificate perform the following steps:

1. clone this repository and checkout main
1. obtain the private key `hanzozt.rootCA.rsa.key`, sha256sum: e8de652fc6fb6b6189bb6de3162d3c3ea5ef019b1e80f6a94ed93f4bdd9d579f
1. cd to the root of the repo
1. create a new branch for the change you want to push 
1. issue the following command and optionally supply a valid `crl_reason`. [rfc5280, section 5.3.1](https://datatracker.ietf.org/doc/html/rfc5280#section-5.3.1)
   
        openssl ca \
            -config ./hanzozt.openssl.conf \
            -keyfile hanzozt.rootCA.rsa.key \
            -cert certs/hanzozt.rootCA.rsa.pem \
            -revoke certs/hanzozt.signing.2021.rsa.pem \
            -crl_reason keyCompromise

1. sign the crl using the private key and generate the actual CRL

       openssl ca -config ./hanzozt.openssl.conf -gencrl -out hanzozt.crl.pem

1. some platforms might DER encoding - export the crl as DER

       openssl crl -in hanzozt.crl.pem -inform PEM -outform DER -out hanzozt.crl

1. commit and push the changes and issue PR to merge to main. Make sure to commit the following files:
    
        serial.num.txt
        certdb.txt
        hanzozt.crl (DER formatted binary crl)
        hanzozt.crl.pem (PEM formatted textual crl)
        

### Revocation Sanity Check

To verify the revoke command has succeeded you should notice a change to the certdb.txt file. The row you have revoked should
change from V for Valid, to R for Revoked. The second field of the file should now show a timestamp and the reason for which
the certificate was revoked. Example below:

    -V      220603050220Z           1004    unknown /CN=Code Signing Certificate .....
    +R      220603050220Z   210602113249Z,keyCompromise     1004    unknown /CN=Code Signing Certificate

## Testing A Revocation

The following commands were issued to generate a certificate which was then revoked:

    openssl genrsa -out hanzozt.signing.rsa.key.torevoke 4096

    openssl req \
        -new -key hanzozt.signing.rsa.key.torevoke \
        -config ./hanzozt.openssl.conf \
        -subj "/CN=RevokeTest hanzozt.org Code Signing Certificate 2021/O=hanzozt.org Inc/OU=adv-dev/C=US/ST=NC" \
        -out hanzozt.signing.rsa.csr.torevoke
    
    openssl ca \
        -batch \
        -config ./hanzozt.openssl.conf \
        -keyfile hanzozt.rootCA.rsa.key \
        -cert certs/hanzozt.rootCA.rsa.pem \
        -days 1098 \
        -in hanzozt.signing.rsa.csr.torevoke \
        -extfile hanzozt.signing.rsa.conf \
        -out certs/hanzozt.signing.2021.rsa.pem.torevoke

    openssl ca \
        -config ./hanzozt.openssl.conf \
        -keyfile hanzozt.rootCA.rsa.key \
        -cert certs/hanzozt.rootCA.rsa.pem \
        -revoke certs/hanzozt.signing.2021.rsa.pem.torevoke \
        -crl_reason keyCompromise

    openssl pkcs12 \
        -export \
        -inkey hanzozt.signing.rsa.key.torevoke \
        -in certs/hanzozt.signing.2021.rsa.pem.torevoke \
        -out hanzozt.signing.rsa.pfx.torevoke
