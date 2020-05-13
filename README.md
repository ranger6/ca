# ca
A simple set of CA (certificate authority) tools. The underlying functionality is provided mostly by OpenSSL.

The tools support setting up and maintaining a Certificate Authority ("CA"), generating Certificate Signing Requests ("CSR"), publishing issued certificates in a registry, and provisioning sites with certificate bundles.  

## contents
The "tools" directory contains various tools and scripts to perform CA tasks.  The tools include their own documentation. See "tools/README.md".

The "tools/conf" directory contains a prototype openssl.cnf file: openssl.cnf.template.  Also provided is a prototype script/configuration file that expands templates: openssl.cnf.cnf.  The top-level conf directory is where one might copy the files from tools/conf and then edit them as desired prior to creating a certificate authority directory. See "initca" and the documentation in the tools directory.

The registry layout is described in the [ranger6/xanna](https://github.com/ranger6/xanna) repo.  The xANNA repo contains a prototype web site for an "Assigned Names and Numbers Authority" with documentation and a registry placeholder.

Remember: the organization/layout of the CA registry is entirely different from the organization/layout of the CA itself.
