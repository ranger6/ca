Tools to Support a Private Certificate Authority
================================================

# Introduction

This note documents tools developed and used to set up a CA and issue/manage certificates.  The work is largely modeled on the (excellent) tutorial covering setting up a CA using openssl: https://jamielinux.com/docs/openssl-certificate-authority/

This note describes a workflow for setting up a CA directory structure, issuing certificates using a few tools, a naming convention for certificates and the registry, and publication to a registry.

# CA Structure

The structure of the filesystem hierarchy used by the CA is based closely on the above mentioned tutorial.  The hierarchy mirrors the certificate trust hierarchy.  Hence the top level contains files used to build the root certficate.  Within this top level directory are sub-directories for each intermediate CA. These sub-directories have the same structure recursively.

## Security

Clearly, the CA structure should be protected from tampering and unauthorized use.  As well, private information (keys) must not be divulged. The CA structure can be (should be) kept under version control with off-site replication.  Git is used here.

All private information is kept in directories named "private".  These should be ignored (via ".gitignore") by git.  Hence, this private information must be provided after checking out a CA structure.  Note as well that only the signing CA's key needs to be present to generate a certificate: the keys up the chain (especially the root) are not needed.

## Naming of CA's, CA Certificates, CN's, and Directories

As a general principle, we try to keep naming coherent between these
entities.  Certain conventions are used to help usage (handling white space, verbosity, case (in)sensitivity, etc ..)

1. Each root CA is given a short name that also serves as the top level directory name.  For example: "ranna". Note that lower case is used for the directory name, but "RANNA" is often capitalized as it is an acronym.  In fact, the CA directory names are not used to identify or name certificates, so no harm done.
2. Each intermediate CA is given a short name that also serves as the sub-directory name.
3. The root CA certificate has a name composed of the CA name plus ".rootCA.cert.pem".  For example: "RANNA.rootCA.cert.pem". The tag "rootCA" marks this certificate as the, well, "root".
4. The intermediate CA certificate names are composed of the root CA name, the chain of intermediate names, and the string ".cert.pem". The name contains the certificate chain up to the root. Again, this convention is loose and does not need to be strictly followed.
5. The common names should mention the CA name and be descriptive.  They do not need to repeat the chain of trust.

## Server and Client (Leaf) Certificate Names

Server certificates are named according to their reverse Domain Name with the string ".cert.pem" appended.  For example, the host "muir" might have a name "fr.remulac.muir.cert.pem".  Note that this is the name used in the CA directory structure. Registries (described below) use a different scheme.  

Client certificates use a variant: a user or account name is prepended to the domain name. This is the format of email addresses or `ssh` addresses. TBD: what does this look like?

# Registry Structure

The CA structure is for internal use by the CA administrators.  Registries are public.  The usage differs as well: registries are for looking up and retrieving certificates; a CA is for production and maintenance of certificates.

1. CA certificates are available at the top level using the Common Name of the CA.  A given CA may have several certificates over time.  These are kept in sub-directories named by each certificate's "subject hash".  In this sub-directory one finds the certificate itself with a fixed name "cert.pem".  A sha256 hash is provided as well.  For example, an ssl CA certificate might be: `RANNA SSL Certificate Authority/cert/1051008f/cert.pem`
2. Server certificates are denoted by their host name (also used to produce the certificate name in the CA structure).  The DNS host name is converted to a file hierarchy.  Under this directory is, again, another directory named by the "subject hash" of the certificate.  Under this directory is the certificate named "cert.pem".  For example: `fr/remulac/muir/cert/75d3e061/cert.pem`.  Note that, in the registry, the chain of trust is not used for CA or other certificate lookup.

More detailed documentation of registry layout can be found in the [xANNA project "registry-doc"](https://github.com/ranger6/xanna).

The coherence between names is such that it is possible to automatically build a certificate chain up from any certificate *up* to the root certificate by following the chain from the target certificate using "Issuer" information within the certificate. This is true even through the registry does not represent trust relationships in the file structure (as opposed to the CA structure).

# Certificate Authorities

Setting up the CA structure is a semi-automated process using the tools described below.  Below is a prolonged example of setting up a CA named "RANNA-CA".  The process follows the tutorial steps.  The tutorial URL's for each step provide the appropriate reference.  The results and variations from the tutorial are covered below.

There are a few simple scripts that capture the steps mostly taken from the tutorial but with the described changes and naming included.  Find them in the "example" directory.

The best way to follow the workflow is to follow both the tutorial and this note in parallel ("Like man, in two windows! Or, three: open up a shell and do it!").  

## Naming conventions in this example

In the example below we are building out a CA and populating a registry.  Assume our CA and "Assigned Names and Number Authority" is called "RANNA".

We'll create the CA in a directory called "ranna-ca".  It's a good idea to keep this under source code control (with appropriate care of private information such as keys).  The CA is not published. Although the CA can be built anywhere, we'll start by extending a git clone of [ranger6/ca](https://github.com/ranger6/ca).

The registry will be a clone of the [ranger6/xanna](https://github.com/ranger6/xanna) repo into "ranna".  The forked repo will have the web content edited appropriately and the registry will be found in "ranna/registry".

Of course, the tools we are using will be from [ranger6/ca](https://github.com/ranger6/ca). We'll put the `tools/bin` directory on our PATH.

## https://jamielinux.com/docs/openssl-certificate-authority/create-the-root-pair.html

The CA is started by cloning the ca git repo and making copies of the template and parameter files:

```Shell
$ git clone https://github.com/ranger6/ca.git ranna-ca
$ cd ranna-ca
$ export PATH="$PATH:$(pwd)/tools/bin"
$ cp tools/conf/* conf
$ ls
LICENSE		README.md	conf/		tools/
$ 
```

The openssl.cnf file is tweaked a bit (via the template):

1. Make a copy of the openssl.cnf.template and openssl.cnf.cnf files to edit.
2. Supply the real name and address information.
3. "dir" will be set to the actual directory ("ranna-ca/ca") when we run `initca`.
4. Some optional info has been removed in the template file (problems with other tools?); be more relaxed about matching state/province names.
5. A new property is introduced to avoid duplication: "ca_prefix". One sees how this matches our naming convention. It will be set when we run `initca`.
6. Some parts of the template are delimited by double braces: substitution will take place when `initca` processes the template.
7. Edit the `openssl.cnf.cnf` file to define values to be substituted when the template is processed.

Here's the diff between the tutorial (`root-config`) and our template file (`openssl.cnf.template` in this example:

```Diff
1,2c1
< # OpenSSL root CA configuration file.
< # Copy to `/root/ca/openssl.cnf`.
---
> # OpenSSL CA configuration file.
10c9,10
< dir               = /root/ca
---
> dir               = {{DIR}}
> ca_prefix	  = {{CA_PREFIX}}
19,20c19,20
< private_key       = $dir/private/ca.key.pem
< certificate       = $dir/certs/ca.cert.pem
---
> private_key       = $dir/private/$ca_prefix.key.pem
> certificate       = $dir/certs/$ca_prefix.cert.pem
24c24
< crl               = $dir/crl/ca.crl.pem
---
> crl               = $dir/crl/$ca_prefix.crl.pem
28c28
< # SHA-1 is deprecated, so use SHA-2 instead.
---
> # SHA-1 is deprecated, so use SHA-256 instead.
35c35
< policy            = policy_strict
---
> policy            = {{POLICY}}
41c41
< stateOrProvinceName     = match
---
> stateOrProvinceName     = optional
64c64
< # SHA-1 is deprecated, so use SHA-2 instead.
---
> # SHA-1 is deprecated, so use SHA-256 instead.
81,86c81,86
< countryName_default             = GB
< stateOrProvinceName_default     = England
< localityName_default            =
< 0.organizationName_default      = Alice Ltd
< organizationalUnitName_default  =
< emailAddress_default            =
---
> countryName_default             = FR
> stateOrProvinceName_default     =
> localityName_default            = Remulac
> 0.organizationName_default      = remulac
> #organizationalUnitName_default  =
> #emailAddress_default            =
105,106d104
< nsCertType = client, email
< nsComment = "OpenSSL Generated Client Certificate"
115,116d112
< nsCertType = server
< nsComment = "OpenSSL Generated Server Certificate"
```

With the `openssl.cnf.template` and `.cnf.cnf` files edited, the root ca directory is created using `initca`:

```Shell
$ initca -t conf/openssl.cnf.template -c conf/openssl.cnf.cnf ./ca RANNA.rootCA
$ ls ca
certs/			csr/			openssl.cnf		private/
crl/			index.txt		openssl.cnf.cnf*	serial
crlnumber		newcerts/		openssl.cnf.template
$
```

We can diff the generated `openssl.cnf` with the template file and see how the text was substituted:

```Diff
$ diff ca/openssl.cnf.template ca/openssl.cnf
9,10c9,10
< dir               = {{DIR}}
< ca_prefix	  = {{CA_PREFIX}}
---
> dir               = /Users/robert/root/share/git/ranna-ca/ca
> ca_prefix	  = RANNA.rootCA
35c35
< policy            = {{POLICY}}
---
> policy            = policy_strict
$
```

The root key is generated using the password "secretpassword".  We are going to name
the key file according to our naming conventions so we don't get mixed up (the tutorial uses simpler names).

```Shell
$ cd ranna
$ openssl genrsa -aes256 -out private/RANNA.rootCA.key.pem 4096
$ chmod 400 private/RANNA.rootCA.key.pem
$ ls -l private 
total 8
-r--------  1 robert  staff  3326 Dec  8 15:58 RANNA.rootCA.key.pem
$
```

The root certificate is created as per the tutorial except that, again, we use our naming convention.
The common name is "RANNA Root Certificate Authority".

```Shell
$ openssl req -config openssl.cnf \
    -key private/RANNA.rootCA.key.pem \
    -new -x509 -days 7300 -sha256 -extensions v3_ca \
    -out certs/RANNA.rootCA.cert.pem
$ chmod 444 certs/RANNA.rootCA.cert.pem
$ ls -l certs
total 8
-r--r--r--  1 robert  staff  2098 Dec  8 16:05 RANNA.rootCA.cert.pem
$ openssl x509 -noout -text -in certs/RANNA.rootCA.cert.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 17669092427689177962 (0xf53539f2b7f5336a)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=FR, L=Remulac, O=remulac, CN=RANNA Root Certificate Authority/emailAddress=remi@example.com
        Validity
            Not Before: Dec  8 15:05:25 2019 GMT
            Not After : Dec  3 15:05:25 2039 GMT
        Subject: C=FR, L=Remulac, O=remulac, CN=RANNA Root Certificate Authority/emailAddress=remi@example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
...

$ 
```

## https://jamielinux.com/docs/openssl-certificate-authority/create-the-intermediate-pair.html

Here, the intermediate CA is named "sslCA" (rather than "intermediate" in the tutorial).

All CA directories look the same and `initca` is used both for the root CA as well as intermediates.  The difference between CA's are in the openssl.cnf files.  Before running `initca`, the openssl.cnf.cnf file (a copy was put in the root CA directory) is edited, setting POLICY to "policy_loose".

```Shell
$ initca ./sslCA RANNA.sslCA
$ ls sslCA
certs/			csr/			openssl.cnf		private/
crl/			index.txt		openssl.cnf.cnf*	serial
crlnumber		newcerts/		openssl.cnf.template
```

Now we create the sslCA key and Certificate Signing Request (CSR):

```Shell
$ cd sslCA
$ openssl genrsa -aes256 -out private/RANNA.sslCA.key.pem 4096
$ chmod 400 private/RANNA.sslCA.key.pem
$ openssl req -config openssl.cnf \
    -key private/RANNA.sslCA.key.pem \
    -new -sha256 \
    -out csr/RANNA.sslCA.csr.pem
```

Create the certificate:

```Shell
$ cd ..		# root ca will issue sslCA cert
$ openssl ca -config openssl.cnf -extensions v3_intermediate_ca \
    -days 3650 -notext -md sha256 \
    -in sslCA/csr/RANNA.sslCA.csr.pem \
    -out sslCA/certs/RANNA.sslCA.cert.pem
$ chmod 444 sslCA/certs/RANNA.sslCA.cert.pem
```

View and and verify it:

Of course, this time, the sslCA's certificate
gets signed by the root CA.  The Common Name is "RANNA SSL Certificate Authority". Here is the sslCA:

```
$ cert sslCA/certs/RANNA.sslCA.cert.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4096 (0x1000)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=FR, L=Remulac, O=remulac, CN=RANNA Root Certificate Authority/emailAddress=remi@example.com
        Validity
            Not Before: Dec  8 15:21:22 2019 GMT
            Not After : Dec  5 15:21:22 2029 GMT
        Subject: C=FR, O=remulac, CN=RANNA SSL Certificate Authority/emailAddress=remi@example.com
...

$ openssl verify -CAfile certs/RANNA.rootCA.cert.pem sslCA/certs/RANNA.sslCA.cert.pem
sslCA/certs/RANNA.sslCA.cert.pem: OK
$
```

You may have noticed that the tool "cert" was used to produce the above output.  This tool is shorthand for listing certificate information: saves a bit of typing.  "cert" and other tools are described below in more detail.

We don't generate the "certificate chain file" as per the tutorial here.  This is handled when we generate server certificates and discussed below.

## Publishing CA Certificates

Now that the root and intermediate CA certificates have been generated, it is time to publish them to one or more registries.  Below, we'll publish to a local registry: `/Users/robert/root/share/git/ranna/registry/ca`.  

The tool "pubcert" does the work of generating the full path name in the registry for the certificate that follows the naming conventions. Unless the "dryrun" (-n) flag is set, the certificate is copied to the target location along with a checksum. The "verbose" (-v) flag echos the name on stdout.

From the top level CA directory, publish the root CA certificate that we generated above:

```Shell
$ pubcert -v /Users/robert/root/share/git/ranna/registry/ca certs/RANNA.rootCA.cert.pem
/Users/robert/root/share/git/ranna/registry/ca/RANNA Root Certificate Authority/cert/4f1c62c5/cert.pem
$
```

We can check the result:

```Shell
$ cd /Users/robert/root/share/git/ranna/registry/ca
$ ls -lR
total 0
drwxr-xr-x  3 robert  staff  96 Dec  8 16:28 RANNA Root Certificate Authority/

./RANNA Root Certificate Authority:
total 0
drwxr-xr-x  3 robert  staff  96 Dec  8 16:28 cert/

./RANNA Root Certificate Authority/cert:
total 0
drwxr-xr-x  4 robert  staff  128 Dec  8 16:28 4f1c62c5/

./RANNA Root Certificate Authority/cert/4f1c62c5:
total 16
-r--r--r--  1 robert  staff  2098 Dec  8 16:28 cert.pem
-rw-r--r--  1 robert  staff    83 Dec  8 16:28 cert.pem.sha256
$
```

The sslCA certificate is likewise published:

```Shell
$ cd /Users/robert/root/share/git/ranna-ca/ca/sslCA
$ pubcert -v /Users/robert/root/share/git/ranna/registry/ca certs/RANNA.sslCA.cert.pem
/Users/robert/root/share/git/ranna/registry/ca/RANNA SSL Certificate Authority/cert/33f2054c/cert.pem
$
```

Note that the intermediate CA certificate is published at the top level of the registry and not
under the root CA hierarchy (as explained above):

```Shell
$ ls -l /Users/robert/root/share/git/ranna/registry/ca
total 0
drwxr-xr-x  3 robert  staff  96 Dec  8 16:28 RANNA Root Certificate Authority/
drwxr-xr-x  3 robert  staff  96 Dec  8 16:31 RANNA SSL Certificate Authority/
$
```

## Signing the CA certificates

As each certificate in the chain of trust is signed by its predecessor except for the root, only the root certificate is missing an authenticating signature: it is self-signed.  One should trust the root certificate only if it is somehow otherwise authenticated. This is sort of a chicken-and-egg problem.  One way to increase trust is to sign the root certificate with a GPG private key.  A good place to put this signature is in the registry in the same directory as the certificate.

# Server and client Certificates

With the CA's set up, one can begin issuing certificates. This note does not cover client certificates (covered in the tutorial).  There are not many differences.

## https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html

The example below covers creating a certificate for a web server where we don't password protect
the certificate key (as explained in the tutorial).

Since creating and publishing certificates risks to be a repetitive process, three tools
are available to simplify the process, reduce errors, and enforce the naming conventions:

1. `mkcsr` to generate keys and "certificate signing requests"
2. `mkcert` to generate and sign a certificate based on a CSR
3. `pubcert` to publish the certificate (not the key!) to a registry

These tools pretty much follow the process outlined in the tutorial.

The creation of a CSR does not depend on the CA structure. In fact, normal usage is for the CSR to be generated and then sent to the CA; requesting a signed certificate (hence, the name "Certificate Signing Request").

The mkcsr tool can be modified to edit a default subject string (see the script).  A default is useful so that clients requesting certificates don't have to work out the complex syntax and use shared conventions. A subject string can be provided as an argument to override the default.

To create a CSR using the default subject string for `muir.remulac.fr`:

```Shell
$ pwd
/Users/robert/root/tmp/csr
$ mkcsr muir.remulac.fr
Generating a 2048 bit RSA private key
.........+++
.............................+++
writing new private key to 'muir.remulac.fr.key.pem'
-----
Certificate Request:
    Data:
        Version: 0 (0x0)
        Subject: C=FR, ST=(none), L=(none), O=remulac, OU=(none), CN=muir.remulac.fr
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d5:d2:7d:6f:42:ff:42:0c:80:5b:b1:18:43:73:
...

$ ls -l
total 16
-rw-r--r--  1 robert  staff  1009 Dec  8 16:38 muir.remulac.fr.csr.pem
-r--------  1 robert  staff  1704 Dec  8 16:38 muir.remulac.fr.key.pem
$
```

Now, the sslCA administrator (after careful examination of the CSR and following lots of other security procedures) generates a certificate and signs it.  This is done with the "mkcert" tool.  By default, this tool expects to be run in the CA's directory, using the openssl.cnf file, finding the CSR in the "csr" directory (using the naming conventions), and putting the certificate in the "certs" directory.  Only the "Common Name" needs to be provided.

If a "registry" is provided, then a verification of the certificate trust chain is performed.  We'll provide this argument and use the defaults for everything else:

```Shell
$ cd /Users/robert/root/share/git/ranna-ca/ca/sslCA
$ cp /Users/robert/root/tmp/csr/muir.remulac.fr.csr.pem csr
$ ls -l csr
total 16
-rw-r--r--  1 robert  staff  1724 Dec  8 16:16 RANNA.sslCA.csr.pem
-rw-r--r--  1 robert  staff  1009 Dec  8 16:42 muir.remulac.fr.csr.pem
$ mkcert muir.remulac.fr -registry /Users/robert/root/share/git/ranna/registry/ca
Using configuration from ./openssl.cnf
Enter pass phrase for /Users/robert/root/share/git/ranna-ca/ca/sslCA/private/RANNA.sslCA.key.pem:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 4096 (0x1000)
...

Certificate is to be certified until Dec 17 15:45:21 2020 GMT (375 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4096 (0x1000)

...

./certs/muir.remulac.fr.cert.pem: OK
$
```

The final "OK" means that "mkcert" built the complete trust chain (with certificates fetched from the registry) and verified the new certificate against it (see the script).

Now publish the certificate:

```Shell
$ pubcert -v /Users/robert/root/share/git/ranna/registry/ca certs/muir.remulac.fr.cert.pem
/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir/cert/c3094fd7/cert.pem
$ ls -l /Users/robert/root/share/git/ranna/registry/ca
total 0
drwxr-xr-x  3 robert  staff  96 Dec  8 16:28 RANNA Root Certificate Authority/
drwxr-xr-x  3 robert  staff  96 Dec  8 16:31 RANNA SSL Certificate Authority/
drwxr-xr-x  3 robert  staff  96 Dec  8 16:48 fr/
$ ls -R /Users/robert/root/share/git/ranna/registry/ca/fr
remulac/

/Users/robert/root/share/git/ranna/registry/ca/fr/remulac:
muir/

/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir:
cert/

/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir/cert:
c3094fd7/

/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir/cert/c3094fd7:
cert.pem		cert.pem.sha256
$ 
```

### Deploying the certificate

The Certificate Authority does not need to be involved with the use/deployment of the certificate.  All the needed information is in the registry(s).  The root CA certificate can be installed in browsers or applications.  Chains of trust from the root or intermediate authority can be built from the registry.  Different applications may have different requirements for configuring certificates and chains. Of course, the server and client keys are neither in the CA nor the registries.

There are a few tools used to build certificate chains.  The functionality and interfaces differ depending on the supported use case and/or preferences.  They are covered below in more detail.

# Tool Summary

The following tools (all bash scripts) are part of the "ca" git project (in the `tools/bin` directory):

* initca -- initialize a certificate authority directory and openssl.cnf file
* cert -- list a certificate or particular fields
* mkkey -- generate an encrypted key pair defaulting to 2048 bits
* mkcsr -- generate a Certificate Signing Request
* mkcert -- generate a signed certificate from a CSR
* rvcert -- revoke a certificate issued by this CA
* mkcrl -- generate a certificate revocation list
* pubcert -- publish a certificate into a registry (or just list its registry pathname)
* chain -- generate a list of registry pathnames in the trust chain from a certificate to the root
* urlchain -- generate a list of registry URL's from a prefix plus a list of certificates
* bldchain -- generate a concatenation of certificates in the trust chain up to the root
* bldbundle -- generate a concatenation of certificates/keys from file names plus URL's

These scripts need to be on the PATH.

The scripts all have descriptions and usage examples in the code.  As well, there is fair degree of error checking and reporting.  It will not be repeated here.

The tools are meant to reduce repetition and use/enforce naming conventions (especially for registries).  As well, a few complicated manipulations have been coded and tested (sorry, no automated tests yet).  In particular, the chaining of certificates by looking at certificate information is a big help.

However, problems may occur that will require an understanding of what is going on.  And, `openssl` is not the simplest tool in the world.  For example, if you make an error in the openssl.cnf file, you'll need to debug it.  Another example: openssl does not want to generate duplicate certificates.  When it complains, you need to look at openssl rather than the scripts.  In short, the implementation is exposed.  These tools are shortcuts and wrappers, not applications.

## CA versus consumer tools (TODO)

Certain tools make life easier for certificate consumers: for example, generating certificate requests and building bundles.  Other tools are strictly for CA administrators.  As of now, all the tools are in the same repo.  It should be easier to make the consumer tools separately available (e.g. published with the registry).

## Key tool dependencies

The tools rely on the standard, usual unix utilities (nothing special to install).  As well, `openssl` must be present or installed.  

There are two very small programs that must be present:

* ftop -- converts a fully qualified domain name to a filesystem path, reversing order
* urlencode -- converts a raw url string (without a query) to an escaped version

These commands, written in `go`, are available on github at [ranger6/ftop](https://github.com/ranger6/ftop) and  [ranger6/urlencode](https://github.com/ranger6/urlencode).  They should be
downloaded, compiled, and installed (i.e. on your PATH).

## Remote vs. local registries

Many of these tools access a local directory and/or compute filenames based on a local filesystem.  A couple of scripts help using URL references to registries, but none write to remote registries.

The general model is the following:

The CA's and the local registry are maintained as git repositories (do not need to be the same repo).  These are the authoritative instances.  Remote registries (e.g. on a web server) are replicas built by copying over the local registry, or better: git pulling the registry.

## initca -- initialize a certificate authority directory and openssl.cnf file

`usage: initca [-f] [-c config] [-t template] location ca_prefix`

Target use: put the simple manual steps of setting up a CA directory into a script. Enable reuse of openssl.cnf files between root CA's and intermediate CA's.

Example usage: see above.

The script creates all the standard directories and files for a CA.  As well, it expands an `openssl.cnf` template (using `sed` and `bash`) file so that it is adapted to the newly created directory.  Other fields can also be defined.  

The prototype (starting point) template and value-definition files can be found in `tools/conf`. You might want make copies of these files prior to editing them (to preserve the original files). The top level `conf` directory is the suggested location.

## cert -- list a certificate or particular fields

`usage: cert [-cn|-serial|-cert|-issuer|-subject_hash|-issuer_hash] cert-name`

Target use: internal tool and visability for any human interested in certificate contents.

Example usage: see above.

This script is used internally by the other scripts.  It is useful for humans wanting to list/examin certificates.

## mkkey -- generate an encrypted key pair defaulting to 2048 bits

`usage: mkkey path [<bits>]`

Target use: shorthand for "genrsa" key generation.

The key is written to `path`.  The user is prompted for a pass-phrase.  The key
can be provided to `mkcsr` (see below).

## mkcsr -- generate a Certificate Signing Request

`usage: mkcsr common-name [-key <file>] [-subj <subject string>]`

Target use: provide clients an easy way to generate CSR's

Example usage: see above.

The script has a default subject string.  It may not be appropriate for your uses. Since the subject string is complicated, it is likely best to define a new default in another version of the script rather than rely on users getting it right.

While the common-name can be anything (that can be used as a file name .. no slashes!), it really is best if the naming convention for DNS names or email/ssh addresses be used.

If no key is provided, a new unencrypted key is generated.

## mkcert -- generate a signed certificate from a CSR

```
usage: mkcert common-name \
    [-config config-file] \
    [-in csr-file] \
    [-out cert-file] \ 
    [-registry registry] \
    [-client] 
```

Target use: CA administrator issuing certificates

Example usage: see above.

The script is meant to be run in a CA directory: the specific CA being used to generate the certificate. This is most likely *not* the *root CA directory*.  See the script to see how the environment can be used to change this.  But note: the openssl.cnf file needs to be consistent with the environment and arguments: just run in the CA directory!

The registry is used to verify the generated certificate.  If the verification fails it is likely because the registry is broken or some other inconsistency has crept in.  If the verification fails, you should re-check the arguments.  After that, you are debugging your CA structure, the registry, the generated certificates/keys, and openssl; not the script.

## rvcert -- revoke a certificate issued by this CA

```
usage: rvcert common-name [-config config-file] [-cert cert-file]
```

Target use: CA administrator managing certificates

The script sort of "undoes" the work of `mkcert`.

The script is meant to be run in a CA directory: the specific CA used to generate the certificate in the first place. A Certificate Revocation List is not generated: only the CA internal database is updated.

## mkcrl -- generate a certificate revocation list

```
usage: mkcrl [-conf conf-file] [-crl prefix]
```

Target use: CA administrator managing certificates

The script is meant to be run in a CA directory: the specific CA used to generate the certificate in the first place.  The "prefix" is the same as the "ca_prefix" used by `initca`.  Since initca places a file in the CA directory with this value, it is generally not necessary to supply this argument.  The CRL is placed in the `crl` directory.

## pubcert -- publish a certificate into a registry (or just list its registry pathname)

`usage: pubcert [-v][-n][-i] registry cert-name`

Target use: CA administrator publishing certificates and as an internal tool.

Example usage: see above.

This turns out to be the most used script in the tool set: not by users, but internally.  When given the file name of a certificate (cert-name) and the path to the top-level of a registry, it computes the canonical certificate path; then copies it there.  The internal use, however, is not to actually copy certificates, but rather to compute the certificate pathname including the pathname of the issuing CA.  This is the step that allows chains of trust to be computed from a registry.

## chain -- generate a list of registry pathnames in the trust chain from a certificate to the root

`usage: chain registry cert-name`

Target use: internal tool.

This is the algorithm that computes a chain of trust (using `pubcert -i`).  It is usable by humans see a list of filenames; but is mostly used by the other scripts below.

It is important to realize that the cert-name is a certificate filename.  That certificate does not need to be in the registry (often it is not).  The certificate file is used to compute the registry location of the certificate: it will be the same file contents, but likely with a different name.

Here is an example (continued from above):

```Shell
$ cd /Users/robert/root/share/git/ranna-ca/ca/sslCA
$ chain /Users/robert/root/share/git/ranna/registry/ca certs/muir.remulac.fr.cert.pem
/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir/cert/c3094fd7/cert.pem
/Users/robert/root/share/git/ranna/registry/ca/RANNA SSL Certificate Authority/cert/33f2054c/cert.pem
/Users/robert/root/share/git/ranna/registry/ca/RANNA Root Certificate Authority/cert/4f1c62c5/cert.pem
$
```

Note that `.../c3094fd7/cert.pem` is the published copy of `certs/muir.remulac.fr.cert.pem`

## urlchain -- generate a list of registry URL's from a prefix plus a list of certificates

`usage: urlchain URL-prefix`

Target use: enable distribution of certificate URL's.

This is a simple script that generates URL's for remote registry references.  There is no access to the remote registry. Unlike "chain", urlchain does not traverse registries: it accepts a sequence of certificate pathnames and then uses these certificates to compute what a remote URL would be. One can provide any list of certificates, including one produced by "chain":

```Shell
$ chain /Users/robert/root/share/git/ranna/registry/ca certs/muir.remulac.fr.cert.pem | \
	urlchain "http://muir.remulac.fr/ranna/registry/ca"
https://muir.remulac.fr/ranna/registry/ca/fr/remulac/muir/cert/c3094fd7/cert.pem
https://muir.remulac.fr/ranna/registry/ca/RANNA%20SSL%20Certificate%20Authority/cert/33f2054c/cert.pem
https://muir.remulac.fr/ranna/registry/ca/RANNA%20Root%20Certificate%20Authority/cert/4f1c62c5/cert.pem
$ 
```

Note the url encoding. The most interesting use case for urlchain is not to create a chain, but simply to generate a URL from a certificate you might have (e.g. a CA certificate).

The "URL-prefix" does not need to be a URL at all: it could be anything including a path prefix.

## bldchain -- generate a concatenation of certificates in the trust chain up to the root

`usage: bldchain registry cert-name`

Target use: provide clients (client apps) with needed certificates to trust.

"bldchain" is used provision applications and clients with certificate chains up to the root.  These chains *do not* need to start at a leaf certificate: they may start at any point in the chain.

Unlike "chain" or "urlchain", "bldchain" outputs the certificate contents and not a list of file names or URLs.

As one might expect from the arguments, "bldchain" uses "chain" internally.

## bldbundle -- generate a concatenation of certificates/keys from file names plus URL's

`usage: bldbundle [file ..][-][file ..]`

Target use: support provisioning of servers that need certificates plus their key

This is the most confusing tool and the most flexible means of building a bundle of certificates and keys (see the tutorial) according to the needs of the application.  The script itself contains quite a bit of explanation.

The motivating use case is something like:
> As a web server admin, I have generated a key and submitted a CSR. The CA has generated a certificate, published it to a registry, and returned to me a list of certificate URLs (perhaps using "urlchain") starting from my new certificate up to the root.  I need to build a bundle of all these certificates with my key file appended.

So, in this example, the list of URLs (on standard input) plus the key file are provided to "bldbundle" which produces a file (the bundle) containing the certificates plus the key:
```Shell
$ cd /Users/robert/root/tmp/csr
$ ls
chainfile.txt			muir.remulac.fr.csr.pem		muir.remulac.fr.key.pem
$ cat chainfile.txt
https://muir.remulac.fr/ranna/registry/ca/fr/remulac/muir/cert/c3094fd7/cert.pem
https://muir.remulac.fr/ranna/registry/ca/RANNA%20SSL%20Certificate%20Authority/cert/33f2054c/cert.pem
https://muir.remulac.fr/ranna/registry/ca/RANNA%20Root%20Certificate%20Authority/cert/4f1c62c5/cert.pem
$ cat chainfile.txt | bldbundle - muir.remulac.fr.key.pem > muir.remulac.fr.bundle.pem
$ chmod 400 muir.remulac.fr.bundle.pem
$ grep BEGIN muir.remulac.fr.bundle.pem
-----BEGIN CERTIFICATE-----
-----BEGIN CERTIFICATE-----
-----BEGIN CERTIFICATE-----
-----BEGIN RSA PRIVATE KEY-----
$ 
```

As expected, we have the three certificates in the chain plus the key file. Of course, an HTTPS server that will serve the resources indicated by the URL's must be available.  That is job of the registry. 
