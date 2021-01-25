---
layout: post
title: "Synology Self-signed Certificate How-to"
description: "Using the Synology NAS Certificates to Provision Private/Locally Scoped Self-signed SSL Certificates"
tags: [yubikey, security, github]
---

It's possible to use a Synology Diskstation's Certificate generation functionality to create a set of privately scoped (non-FQDN) self-signed SSL certificates that you can use to provision internal network services so that connecting to them does not cause your browser to throw warning messages (or in the case of Chrome, prevent you from connecting at all).

# Rationale

Usually, when you add network devices to your personal private network, they are refereneced by IP addresses as naming requires either maintaining individual host files on each machine or setting up DNS. The first is pretty cumbersome; the second seems like overkill (unless you're a masochist, which I have been in the past). As an alternative, I considered using locally scoped names associated with fixed IPs associated via a light-weight DNS resolver (in my case, using `unbound` running on my Raspberry Pi with Pi-Hole).

**WARNING: This is clearly a *HACK* and is not intended to be used for production environments. If you need full SSL certificates on your internal network, you'll need to configure split-horizon DNS with a FQDNs and a certificate provider (probably LetsEncrypt if you're trying to do this for free, which Diskstation supports).**

# Generate the Certificate on the Diskstation

1. **Navigate to `Control Panel` -> `Security` -> `Certificates`.** ![Screen Shot 2020-10-18 at 11 02 56 PM](https://user-images.githubusercontent.com/165323/96403659-234dcd00-1196-11eb-9c89-9e7366b7fa64.png)

1. **Click on `Add`. This should bring up the `Create certificate` dialog. Choose `Add a new certificate` and click `Next`.**

   ![Screen Shot 2020-10-18 at 11 04 30 PM](https://user-images.githubusercontent.com/165323/96403728-52643e80-1196-11eb-8289-cfc37b2a2373.png)

1. **Fill in the `Description` (something which identifies this local cert). Choose the `Create self-signed certificate` option and click `Next`.**

   ![Screen Shot 2020-10-18 at 11 05 20 PM](https://user-images.githubusercontent.com/165323/96403786-6f007680-1196-11eb-8927-fe925687732a.png)

1. **Fill in the root certificate details and click `Next`.**

   ![Screen Shot 2020-10-18 at 11 07 12 PM](https://user-images.githubusercontent.com/165323/96403919-c3a3f180-1196-11eb-99f6-d7d49307fac0.png)

1. **Specify the `Subject Alternative Name` which should match the local non-FQDN name of the machine on your network. Click `Next`.**

   ![Screen Shot 2020-10-18 at 11 07 43 PM](https://user-images.githubusercontent.com/165323/96403926-ca326900-1196-11eb-97ac-b21bb33cb3c8.png)

> **WARNING:** The certificate _MUST_ have a `Subject Alternative Name` which matches the the local non-FQDN name or Chrome (not Firefox) will show an insecure connection when access this machine although https is being used.

# Download the generated certificates and import them into your certificate manager.

1. **Right click on the new certificate and choose the `Export certificate` option. This will download all of the certificates into an `archive.zip` file.**

   ![Screen Shot 2020-10-18 at 11 09 45 PM](https://user-images.githubusercontent.com/165323/96404032-12518b80-1197-11eb-948b-84d06d77fe06.png)

1. **Unpack the `archive.zip` file. This should contain the following files:**

    - `syno-ca-privkey.pem` - The private key for the generated root certificaate
    - `syno-ca-cert.pem`- The public key for the generated root certificate
    - `privkey.pem` - The private key for the generated server certificate (which matches the `Subject Alternative Name`/local non-FQDN)
    - `cert.pem` - The public key for the generated server certificate (which matches the `Subject Alternative Name`/local non-FQDN)

## In MacOS/OSX use Keychain Access to import the certificates

1. **Import the `syno-ca-cert.pem` public key (either by clicking on it or using the `Import Items` option in Keychain Manager.**

> **NOTE:** You will be asked to provide your administrative password as these items are being added to the machine's keystore. _These should be stored in the `System` Keychain._

### Grant the root certificate `Always Trust` privileges

1. **The `syno-ca-cert.pem` is the self-signed root certificate responsible for being the certificate authority for additional certificates. Trusting this certificate means that additional certs which use this as their CA will _automatically_ be trusted.**

1. **To set this to `Always Trust`, select the root certificate in the Keychain Manager and right-click to `Get Info`. This should bring up a dialog box. Note that there are two sections - `Trust` and `Details`. Expand the `Trust` section and in the `when using this certificate` field, select `Always Trust`.**

   ![Screen Shot 2020-10-27](https://user-images.githubusercontent.com/165323/97377417-95549f00-1885-11eb-9e65-092758e1c7ba.png)

### Import the associated SSL certificate for the Diskstation

1. **The `cert.pem` is the self-signed server certificate that can be used to establish `https` connections to the machine. Follow the import procedure you used for the root certificate. Unlike the root certificate, you will _NOT_ need to set the `Trust` on this certificate explicitly as it will be inherited from the root certificate previously configured.**

# Generate another certificate using the previously generated Diskstation root certificate

You can use this procedure to create SSL certificates for other machines on your network.

## Generate another certificate

Follow the instructions above to create a new self-signed certificate. We will be reusing the original the _ORIGINAL_ generated root certificate (`syno-ca-cert.pem`) and private key (`syno-ca-privkey.pem`) to create a **certificate signing request (CSR)** for this new certificate.

## Create a certificate signing request for the generated cert

```console
% openssl 
```

## Use the DSM CSR functionality to sign the new certificate

1. **Navigate to `Control Panel` -> `Security` -> `Certificates` on the DSM and click the `CSR` button.**

1. **In the `Create certificate` dialog box, select the `Sign certificate signing request` option and click `Next`.**

   ![Screen Shot 2020-10-18 at 11 11 09 PM](https://user-images.githubusercontent.com/165323/96404105-4927a180-1197-11eb-81ae-955e13eafb16.png)

   > **NOTE:** Ensure that you use the correct certificate root you generated previously to ensure that the new certificaate is associated with this self-signed CA.

1. **Upload the CSR generated via `openssl` in the previous step. Ensure that the `Subject Alternative Name` matches the local non-FQDN name of the server you are associating with this certificate.**

   ![Screen Shot 2020-10-18 at 11 11 27 PM](https://user-images.githubusercontent.com/165323/96404120-4fb61900-1197-11eb-8999-2bef1120ed1c.png)

1. **Download the signed certificate.**

1. **Import the `cert.pem` into Keychain Manager. Because this certificate was signed by the root certificate already granted `Trust`, this certificate should JUST WORK (fingers crossed).**

# Caveats

- The DSM does _NOT_ allow you to specify the certificate `Validity Period` for the root certificate/orignal cert. Strangely, it does allow you to specify it when using the `CSR` functionality.

# TODO

- [ ] provide the `openssl` command to generate the CSR