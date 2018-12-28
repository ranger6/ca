Tools to Support a Private Certificate Authority
================================================

# Introduction

This note documents the tools developed and used to set up a CA and issue/manage certificates.
The work is largely modeled on the (excellent) tutorial
covering setting up a CA using openssl: https://jamielinux.com/docs/openssl-certificate-authority/

This note describes a workflow for setting up a registry and issuing certificates using a few tools
and a naming convention for certificates and the registry.

# CA Structure

The structure of the filesystem hierarchy used by the CA is based closely on the above mentioned tutorial.
The hierarchy mirrors the certificate trust hierarchy.  Hence the top level contains files used
to build the root certficate.  Within this top level directory are sub-directories for each intermediate
CA. These sub-directories have the same structure recursively.

## Security

Clearly, the CA structure should be protected from tampering and unauthorized use.  As well, private information (keys) must not be divulged. The CA structure can be (should be) kept under version control with off-site replication.  Git is used here.

All private information is kept in directories named "private".  These should be ignored (via ".gitignore") by git.  Hence, this
private information must be provided after checking out a CA structure.  Note as well that only the signing CA's key needs
to be present to generate a server certificate: the keys up the chain (especially the root) are not needed.

## Naming of CA's, CA Certificates, CN's, and Directories

As a general principle, we try to keep naming coherent between these
entities.  Certain conventions are used to help usage (handling white space, verbosity, case (in)sensitivity, etc ..)

1. Each root CA is given a short name that also serves as the top level directory name.  For example: "ranna". Note that lower case is used for the directory name, but "RANNA" is often capitalized as it is an acronym.  In fact, the CA directory names are not used to identify or name certificates, so no harm done.
2. Each intermediate CA is given a short name that also serves as the sub-directory name.
3. The root CA certificate has a name composed of the CA name plus ".rootCA.cert.pem".  For example: "RANNA.rootCA.cert.pem". The tag "rootCA" marks this certificate as the, well, "root".
4. The intermediate CA certificate names are composed of the root CA name, the chain of intermediate names, and the string ".cert.pem". The name contains the certificate chain up to the root. Again, this convention is loose and does not need to be strictly followed.
5. The common names should mention the CA name and be descriptive.  They do not need to repeat the chain of trust.

## Server (Leaf) Certificate Names

Server certificates are named according to their reverse Domain Name with the string ".cert.pem" appended.
For example, the host "muir" might have a name "fr.remulac.muir.cert.pem".
Note that this is the name used in the CA directory structure. Registries (described below) use a different scheme.

# Registry Structure

The CA structure is for internal use by the CA administrators.  Registries are public.
The usage differs as well: registries are for looking up and retrieving certificates;
a CA is for production and maintenance of certificates.

1. CA certificates are available at the top level using the Common Name of the CA.  A given CA may have several certificates over time.  These are kept in sub-directories named by each certificate's "subject hash".
In this sub-directory one finds the certificate itself with a fixed name "cert.pem".  A sha256 hash is provided as well.  For example, an ssl CA certificate might be: `RANNA SSL Certificate Authority/cert/1051008f/cert.pem`
2. Server certificates are denoted by their host name (also used to produce the certificate name in the CA structure).
The DNS host name is converted to a file hierarchy.
Under this directory is, again, another directory named by the "subject hash" of the certificate.
Under this directory is the certificate named "cert.pem".
For example: `fr/remulac/muir/cert/75d3e061/cert.pem`.
Note that, in the registry, the chain of trust is not used for CA or server certificate lookup.

More detailed documentation of registry layout can be found in the [xANNA project "registry-doc"](https://github.com/ranger6/xanna).

The coherence between names is such that it is possible to automatically build a certificate chain up from any
certificate *up* to the root certificate by following the chain from the target certificate using "Issuer" information
within the certificate. This is true even through the registry does not represent trust relationships
in the file structure (as opposed to the CA structure).

# Certificate Authorities

Setting up the CA structure is (as of this writing) a manual process.  As an example, we are setting up a CA named "ranna".  The process follows the tutorial steps.  The tutorial URL's for each step provide the appropriate reference.  The results and variations from the tutorial are covered below.

There are a few simple scripts that capture the steps mostly taken from the tutorial but
with the described changes and naming included.  Find them in the "example" directory.
This is the start of providing a "one line" command to initialize a CA.

The best way to follow the workflow is to follow both the tutorial and this note in parallel ("Like man, in two windows! Or, three: open up a shell and do it!").  

## Naming conventions in this example

In the example below we are building out a CA and populating a registry.  Assume our CA and "Assigned Names and Number Authority" is called "RANNA".

We'll create the CA in a directory called "ranna-ca".  It's a good idea to keep this under source code control (with appropriate care of private information such as keys).  The CA is not published.

The registry will be fork of the [ranger6/xanna](https://github.com/ranger6/xanna) repo into "ranna".  The forked repo will have the web content edited appropriately and the registry will be found in "ranna/registry".

Of course, the tools we are using will be from [ranger6/ca](https://github.com/ranger6/ca)

## https://jamielinux.com/docs/openssl-certificate-authority/create-the-root-pair.html

The root CA directory ("ranna-ca") is set up:

```Shell
% mkdir ranna-ca
% cd ranna-ca
...
%  ls
certs/		crl/		index.txt	newcerts/	private/	serial
```

The openssl.cnf file is tweaked a bit:

1. The real name and address information is supplied.
2. "dir" is set to the actual directory ("ranna-ca").
3. Some optional info is removed (problems with other tools?); be more relaxed about matching state/province names.
4. A new property is introduced to avoid duplication: "ca_prefix". One sees how this matches our naming convention.

Here's the diff between the tutorial and our example:

```Diff
2c2
< # Copy to `/root/ca/openssl.cnf`.
---
> # RANNA Root CA
10c10,11
< dir               = /root/ca
---
> dir               = /Users/robert/root/share/git/ranna-ca
> ca_prefix	    = RANNA.rootCA
19,20c20,21
< private_key       = $dir/private/ca.key.pem
< certificate       = $dir/certs/ca.cert.pem
---
> private_key       = $dir/private/$ca_prefix.key.pem
> certificate       = $dir/certs/$ca_prefix.cert.pem
24c25
< crl               = $dir/crl/ca.crl.pem
---
> crl               = $dir/crl/$ca_prefix.crl.pem
41c42
< stateOrProvinceName     = match
---
> stateOrProvinceName     = optional
81,86c82,87
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
> organizationalUnitName_default  = small town
> emailAddress_default            = ranger6rob@gmail.com
105,106d105
< nsCertType = client, email
< nsComment = "OpenSSL Generated Client Certificate"
115,116d113
< nsCertType = server
< nsComment = "OpenSSL Generated Server Certificate"
```

The root key is generated using the password "secretpassword".  But we are going to name
the key file according to our naming conventions so we don't get mixed up (the tutorial uses simpler names).

```Shell
% ls -l private 
total 8
-r--------  1 robert  staff  3326 Aug  9 18:14 RANNA.rootCA.key.pem
```

The root certificate is created as per the tutorial except that, again, we use our naming convention.
The common name--again, following the naming conventions--is "RANNA Root Certificate Authority".

```Shell
% ls -l certs
total 8
-r--r--r--  1 robert  staff  2285 Aug  9 18:22 RANNA.rootCA.cert.pem
% openssl x509 -noout -text -in certs/RANNA.rootCA.cert.pem 
Certificate:
    Data:
	Version: 3 (0x2)
	Serial Number:
	    db:af:06:46:91:57:e1:3f
	Signature Algorithm: sha256WithRSAEncryption
	Issuer: C=FR, L=Remulac, O=remulac, OU=small town, CN=RANNA Root Certificate Authority/emailAddress=ranger6rob@gmail.com
	Validity
	    Not Before: Aug  9 16:22:26 2017 GMT
	    Not After : Aug  4 16:22:26 2037 GMT
	Subject: C=FR, L=Remulac, O=remulac, OU=small town, CN=RANNA Root Certificate Authority/emailAddress=ranger6rob@gmail.com
	Subject Public Key Info:
	    Public Key Algorithm: rsaEncryption
	    RSA Public Key: (4096 bit)
...
% 
```

## https://jamielinux.com/docs/openssl-certificate-authority/create-the-intermediate-pair.html

Here, the intermediate CA is named "sslCA" (rather than "intermediate" in the tutorial). But, we set
things up as explained: creating a directory under "ranna" with the same CA structure.

```Shell
% ls -l
total 16
drwxr-xr-x  2 robert  staff  68 Aug  9 18:34 certs/
drwxr-xr-x  2 robert  staff  68 Aug  9 18:34 crl/
-rw-r--r--  1 robert  staff   5 Aug  9 18:35 crlnumber
drwxr-xr-x  2 robert  staff  68 Aug  9 18:34 csr/
-rw-r--r--  1 robert  staff   0 Aug  9 18:34 index.txt
drwxr-xr-x  2 robert  staff  68 Aug  9 18:34 newcerts/
drwx------  2 robert  staff  68 Aug  9 18:34 private/
-rw-r--r--  1 robert  staff   5 Aug  9 18:34 serial
```

Because the "ca_prefix" is being used, the diff between the sslCA openssl.cnf file and the root configuration file
is even simpler than in the tutorial:

```Diff
2c2
< # RANNA SSL CA
---
> # RANNA Root CA
10,11c10,11
< dir               = /Users/robert/root/share/git/ranna-ca/sslCA
< ca_prefix	    = RANNA.sslCA
---
> dir               = /Users/robert/root/share/git/ranna-ca
> ca_prefix	    = RANNA.rootCA
36c36
< policy            = policy_loose
---
> policy            = policy_strict
```

Almost the same exercise: create a key, create a CSR, create a certificate, check it.  Of course, this time, the sslCA's certificate
gets signed by the root CA.  The Common Name is "RANNA SSL Certificate Authority". Here is the sslCA:

```
% cert certs/RANNA.sslCA.cert.pem 
Certificate:
    Data:
	Version: 3 (0x2)
	Serial Number: 4096 (0x1000)
	Signature Algorithm: sha256WithRSAEncryption
	Issuer: C=FR, L=Remulac, O=remulac, OU=small town, CN=RANNA Root Certificate Authority/emailAddress=ranger6rob@gmail.com
	Validity
	    Not Before: Aug  9 17:23:01 2017 GMT
	    Not After : Aug  7 17:23:01 2027 GMT
	Subject: C=FR, O=remulac, OU=small town, CN=RANNA SSL Certificate Authority/emailAddress=ranger6rob@gmail.com
...
```

You may have noticed that the tool "cert" was used to produce the above output.  This tool
is shorthand for listing certificate information: saves a bit of typing.  "cert" and other
tools are described below in more detail.

We don't generate the "certificate chain file" as per the tutorial here.  This is handled
when we generate server certificates and discussed below.

## Publishing CA Certificates

Now that the root and intermediate CA certificates have been generated, it is time to publish them to one or
more registries.  Below, we'll publish to a local registry: `/Users/robert/root/share/git/ranna/registry/ca`.

The tool "pubcert" does the work of generating the full path name in the registry for the certificate
that follows the naming conventions. Unless the "dryrun" (-n) flag is set, the certificate is copied to the
target location along with a checksum. The "verbose" (-v) flag echos the name on stdout.

From the top level CA directory, publish the root CA certificate that we generated above:

```Shell
% pubcert -v /Users/robert/root/share/git/ranna/registry/ca certs/RANNA.rootCA.cert.pem 
/Users/robert/root/share/git/ranna/registry/ca/RANNA Root Certificate Authority/cert/1f541a81/cert.pem
%
```

We can check the result:

```Shell
% cd /Users/robert/root/share/git/ranna/registry/ca
% ls -lR .
total 0
drwxr-xr-x  3 robert  staff  102 Aug 10 11:04 RANNA Root Certificate Authority/

./RANNA Root Certificate Authority:
total 0
drwxr-xr-x  3 robert  staff  102 Aug 10 11:04 cert/

./RANNA Root Certificate Authority/cert:
total 0
drwxr-xr-x  4 robert  staff  136 Aug 10 11:04 1f541a81/

./RANNA Root Certificate Authority/cert/1f541a81:
total 16
-rw-r--r--  1 robert  staff  2285 Aug 10 11:04 cert.pem
-rw-r--r--  1 robert  staff    83 Aug 10 11:04 cert.pem.sha256
% 
```

The sslCA certificate is likewise published:

```Shell
% cd /Users/robert/root/share/git/ranna-ca
% pubcert -v /Users/robert/root/share/git/ranna/registry/ca sslCA/certs/RANNA.sslCA.cert.pem 
/Users/robert/root/share/git/ranna/registry/ca/RANNA SSL Certificate Authority/cert/1051008f/cert.pem
%
```

Note that the intermediate CA certificate is published at the top level of the registry and not
under the root CA hierarchy (as explained above):

```Shell
% ls -l /Users/robert/root/share/git/ranna/registry/ca
total 0
drwxr-xr-x  3 robert  staff  102 Aug 10 11:04 RANNA Root Certificate Authority/
drwxr-xr-x  3 robert  staff  102 Aug 10 11:10 RANNA SSL Certificate Authority/
%
```

## Signing the CA certificates

As each certificate in the chain of trust is signed by its predecessor except for the root, only the
root certificate is missing an authenticating signature: it is self-signed.  One should trust the
root certificate only if it is somehow otherwise authenticated. This is sort of a chicken-and-egg
problem.  One way to increase trust is to sign the root certificate with a GPG private key.  A
good place to put this signature is in the registry in the same directory as the certificate.

# Server Certificates

With the CA's set up, one can begin issuing server certificates. This note does not cover
client certificates (covered in the tutorial).  There are not many differences.

## https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html

The example below covers creating a certificate for a web server where we don't password protect
the certificate key (as explained in the tutorial).

Since creating and publishing server certificates risks to be a repetitive process, three tools
are available to simplify the process, reduce errors, and enforce the naming conventions:

1. `csr` to generate keys and "certificate signing requests"
2. `mkcert` to generate and sign a server certificate based on a CSR
3. `pubcert` to publish the certificate (not the key!) to a registry

These tools pretty much follow the process outlined in the tutorial.

The creation of a CSR does not depend on the CA structure. In fact, normal usage
is for the CSR to be generated and then sent to the CA; requesting a signed
certificate (hence, the name "Certificate Signing Request").

The csr tool can be modified to edit a default subject line (see the script).  A default is useful so that clients requesting certificates don't have to work out the complex syntax and use shared conventions.

To create a CSR using the default subject line for `muir.remulac.fr`:

```Shell
% pwd
/Users/robert/root/tmp
% csr muir.remulac.fr
Generating a 2048 bit RSA private key
.......................+++
...........................+++
writing new private key to 'muir.remulac.fr.key.pem'
-----
Certificate Request:
    Data:
	Version: 0 (0x0)
	Subject: C=FR, ST=(none), L=(none), O=remulac, OU=small town, CN=muir.remulac.fr
	Subject Public Key Info:
	    Public Key Algorithm: rsaEncryption
	    RSA Public Key: (2048 bit)
		Modulus (2048 bit):
		    00:ed:fe:3c:bb:79:7c:99:60:54:67:39:3b:45:f0:
...

% ls -l
total 16
-rw-r--r--  1 robert  staff  1078 Aug 10 11:47 muir.remulac.fr.csr.pem
-r--------  1 robert  staff  1679 Aug 10 11:47 muir.remulac.fr.key.pem
%
```

Now, the sslCA administrator (after careful examination of the CSR and following lots of other security procedures) generates a certificate and signs it.  This is done with the "mkcert" tool.  By default, this tool expects to be run in the CA's directory, using the openssl.cnf file, finding the CSR in the "csr" directory (using the naming conventions), and putting the certificate in the "certs" directory.  Only the "Common Name" needs to be provided.

If a "registry" is provided, then a verification of the certificate trust chain is performed.  We'll provide this argument and use the defaults for everything else:

```Shell
% cd /Users/robert/root/share/git/ranna-ca/sslCA
% cp /Users/robert/root/tmp/muir.remulac.fr.csr.pem csr
% ls -l csr
total 16
-rw-r--r--  1 robert  staff  1817 Aug  9 19:09 RANNA.sslCA.csr.pem
-rw-r--r--  1 robert  staff  1078 Aug 10 12:12 muir.remulac.fr.csr.pem
% mkcert muir.remulac.fr -registry /Users/robert/root/share/git/ranna/registry/ca
Using configuration from ./openssl.cnf
Enter pass phrase for /Users/robert/root/share/git/ranna-ca/sslCA/private/RANNA.sslCA.key.pem: xxxxxxxxxxx
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 4098 (0x1002)
...

Certificate is to be certified until Aug 20 11:39:56 2018 GMT (375 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
Certificate:
    Data:
	Version: 3 (0x2)
...
./certs/muir.remulac.fr.cert.pem: OK
%
```

The final "OK" means that "mkcert" built the complete trust chain (with certificates
fetched from the registry) and verified
the new certificate against it (see the script).

Now publish the certificate:

```Shell
% pubcert -v /Users/robert/root/share/git/ranna/registry/ca certs/muir.remulac.fr.cert.pem 
/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir/cert/75d3e061/cert.pem
% ls -l /Users/robert/root/share/git/ranna/registry/ca
total 0
drwxr-xr-x  3 robert  staff  102 Aug 10 11:04 RANNA Root Certificate Authority/
drwxr-xr-x  3 robert  staff  102 Aug 10 11:10 RANNA SSL Certificate Authority/
drwxr-xr-x  3 robert  staff  102 Aug 10 13:49 fr/
% ls -R /Users/robert/root/share/git/ranna/registry/ca/fr
remulac/

/Users/robert/root/share/git/ranna/registry/ca/fr/remulac:
muir/

/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir:
cert/

/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir/cert:
75d3e061/

/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir/cert/75d3e061:
cert.pem		cert.pem.sha256
% 
```

### Deploying the certificate

The Certificate Authority does not need to be involved with the use/deployment of the certificate.
All the needed information is in the registry(s).  The root CA certificate can be installed in
browsers or applications.  Chains of trust from the root or intermediate authority can be built
from the registry.  Different applications may have different requirements for configuring
certificates and chains.

There are a few tools used to build certificate chains.  The functionality and interfaces
differ depending on the supported use case and/or preferences.  They are covered below
in more detail.

# Tool Summary

The following tools (all bash scripts) are part of the "ca" git project (in the `tools/ca/bin` directory):

* cert -- list a certificate or particular fields
* csr -- generate a Certificate Signing Request
* mkcert -- generate a signed certificate from a CSR
* pubcert -- publish a certificate into a registry (or just list its registry pathname)
* chain -- generate a list of registry pathnames in the trust chain from a certificate to the root
* urlchain -- generate a list of registry URL's from a prefix plus a list of certificates
* bldchain -- generate a concatenation of certificates in the trust chain up to the root
* bldbundle -- generate a concatenation of certificates/keys from file names plus URL's

These scripts need to be on the PATH.

The scripts all have descriptions and usage examples in the code.  As well, there is fair degree
of error checking and reporting.  It will not be repeated here.

The tools are meant to reduce repetition and use/enforce naming conventions (especially for registries).
As well, a few complicated manipulations have been coded and tested (sorry, no automated tests yet).  In particular,
the chaining of certificates by looking at certificate information is a big help.

However, problems may occur that will require an understanding of what is going on.  And, `openssl` is not
the simplest tool in the world.  For example, if you make an error in the openssl.cnf file, you'll need
to debug it.  Another example: openssl does not want to generate duplicate certificates.  When it complains,
you need to look at openssl rather than the scripts.  In short, the implementation is exposed.  These
tools are shortcuts and wrappers, not applications.

## CA versus consumer tools (TODO)

Certain tools make life easier for certificate consumers: for example, generating certificate requests and building bundles.
Other tools are strictly for CA administrators.  As of now, all the tools are in the same repo.  It should be
easier to make the consumer tools separately available (e.g. published with the registry).

## Key tool dependencies

The tools rely on the standard, usual unix utilities (nothing special to install).  As well, `openssl` must be present or installed.

There are two very small programs that must be present:

* ftop -- converts a fully qualified domain name to a filesystem path, reversing order
* urlencode -- converts a raw url string (without a query) to an escaped version

These commands, written in `go`, are available on github at [ranger6/ftop](https://github.com/ranger6/ftop) and  [ranger6/urlencode](https://github.com/ranger6/urlencode).  They should be
downloaded, compiled, and installed.

## Remote vs. local registries

Many of these tools access a local directory and/or compute filenames based on a local filesystem.
A couple of scripts help using URL references to registries, but none write to remote registries.

The general model is the following:

The CA's and the local registry are maintained as git repositories (do not need to be the same repo).
These are the authoritative instances.  Remote registries (e.g. on a web server) are replicas built
by copying over the local registry, or better: git pulling the registry.

## cert -- list a certificate or particular fields

`usage: cert [-cn|-serial|-cert|-issuer|-subject_hash|-issuer_hash] cert-name`

Target use: internal tool and visability for any human interested in certificate contents.

This script is used internally by the other scripts.  It is useful for humans wanting to
list/examin certificates.

## csr -- generate a Certificate Signing Request

`usage: csr common-name [ -subj <subject string> ]`

Target use: provide clients an easy way to generate CSR's

See the example above. The script has a default subject string.
It may not be appropriate for your uses. Since the
subject string is complicated, it is likely best to define a new default in another
version of the script rather than rely on users getting it right.

While the common-name can be anything (that can be used as a file name .. no slashes!), it
really is best if the naming convention for DNS names be used.

## mkcert -- generate a signed certificate from a CSR

```
usage: mkcert common-name \
    [ -config config-file ] \
    [ -in csr-file ] \
    [ -out cert-file ] \ 
    [ -registry registry ] 
```

Target use: CA administrator issuing certificates

See the example above.  The script is meant to be run in a CA directory: the specific CA being used
to generate the certificate. This is most likely *not* the *root CA directory*.  See the script
to see how the environment can be used to change this.  But note: the openssl.cnf file
needs to be consistent with the environment and arguments: just run in the CA directory!

The registry is used to verify the generated certificate.  If the verification fails
it is likely because the registry is broken or some other inconsistency has crept
in.  If the verification fails, you should re-check the arguments.  After that, you are
debugging your CA structure, the registry, the generated certificates/keys, and openssl; not
the script.

## pubcert -- publish a certificate into a registry (or just list its registry pathname)

`usage: pubcert [-v][-n][-i] registry cert-name`

Target use: internal tool.

This turns out to be the most used script in the tool set: not by users, but internally.
When given the file name of a certificate (cert-name) and the path to a registry, it
computes the canonical certificate path; then copies it there.  The internal use
is not to actually copy certificates, but rather to compute the certificate pathname
including the pathname of the issuing CA.  This is the step that allows chains of
trusts to be computed from a registry.

## chain -- generate a list of registry pathnames in the trust chain from a certificate to the root

`usage: chain registry cert-name`

Target use: internal tool.

This is the algorithm that computes a chain of trust (using `pubcert -i`).  It is usable by
humans see a list of filenames; but is mostly used by the other scripts below.

It is important to realize that the cert-name is a certificate filename.  That certificate
does not need to be in the registry (often it is not).  The certificate file is used to
compute the registry location of the certificate: it will be the same file contents, but likely with a
different name.

Here is an example (continued from above):

```Shell
% cd /Users/robert/root/share/git/ranna-ca/sslCA
% chain /Users/robert/root/share/git/ranna/registry/ca certs/muir.remulac.fr.cert.pem
/Users/robert/root/share/git/ranna/registry/ca/fr/remulac/muir/cert/75d3e061/cert.pem
/Users/robert/root/share/git/ranna/registry/ca/RANNA SSL Certificate Authority/cert/1051008f/cert.pem
/Users/robert/root/share/git/ranna/registry/ca/RANNA Root Certificate Authority/cert/1f541a81/cert.pem
%
```

`.../75d3e061/cert.pem` is a copy of `certs/muir.remulac.fr.cert.pem`

## urlchain -- generate a list of registry URL's from a prefix plus a list of certificates

`usage: urlchain URL-prefix`

Target use: enable distribution of certificate URL's.

This is a simple script that generates URL's for remote registry references.  There is no
access to the remote registry. Unlike "chain", urlchain does not traverse registries: it accepts
a sequence of certificate pathnames and then uses these certificates to compute what a remote
URL would be. We can provide any list of certificates, including one produced by "chain":

```Shell
% chain /Users/robert/root/share/git/ranna/registry/ca certs/muir.remulac.fr.cert.pem | \
	urlchain "http://muir.remulac.fr/ranna/registry/ca"
http://muir.remulac.fr/ranna/registry/ca/fr/remulac/muir/cert/75d3e061/cert.pem
http://muir.remulac.fr/ranna/registry/ca/RANNA%20SSL%20Certificate%20Authority/cert/1051008f/cert.pem
http://muir.remulac.fr/ranna/registry/ca/RANNA%20Root%20Certificate%20Authority/cert/1f541a81/cert.pem
% 
```

Note the url encoding. The most interesting use case for urlchain is not to create a chain, but
simply to generate a URL from a certificate you might have (e.g. a CA certificate).

The "URL-prefix" does not need to be a URL at all: it could be anything including a path prefix.

## bldchain -- generate a concatenation of certificates in the trust chain up to the root

`usage: bldchain registry cert-name`

Target use: provide clients (client apps) with needed certificates to trust.

"bldchain" is used provision applications and clients with certificate chains up to the root.  These
chains *do not* need to start at a server (leaf) certificate: they may start at any point in the chain.

Unlike "urlchain", "bldchain" outputs the certificate contents and not a list of file names or URLs.

As one might expect from the arguments, "bldchain" uses "chain" internally.

## bldbundle -- generate a concatenation of certificates/keys from file names plus URL's

`usage: bldbundle [file ..][-][file ..]`

Target use: support provisioning of servers that need certificates plus their key

This is the most confusing tool and the most flexible means of building a bundle of certificates
and keys (see the tutorial) according to the needs of the application.  The script itself contains quite a bit of explanation.

The motivating use case is something like:
> As a web server admin, I have generated a key and submitted a CSR. The CA has generated a certificate, published it to a registry, and returned to me a list of certificate URLs (using urlchain) starting from my new certificate up to the root.  I need to build a bundle of all these certificates with my key file appended.

```Shell
% cat chainfile.txt
http://muir.remulac.fr/ranna/registry/ca/fr/remulac/muir/cert/75d3e061/cert.pem
http://muir.remulac.fr/ranna/registry/ca/RANNA%20SSL%20Certificate%20Authority/cert/1051008f/cert.pem
http://muir.remulac.fr/ranna/registry/ca/RANNA%20Root%20Certificate%20Authority/cert/1f541a81/cert.pem
% ls
muir.remulac.fr.csr.pem	muir.remulac.fr.key.pem	chainfile.txt
% cat chainfile.txt | bldbundle - muir.remulac.fr.key.pem > muir.remulac.fr.bundle.pem
% chmod 400 muir.remulac.fr.bundle.pem
% grep BEGIN muir.remulac.fr.bundle.pem
-----BEGIN CERTIFICATE-----
-----BEGIN CERTIFICATE-----
-----BEGIN CERTIFICATE-----
-----BEGIN RSA PRIVATE KEY-----
% 
```

As expected, we have the three certificates in the chain plus the key file.
