+++
draft = false
date = 2019-08-30T07:49:44-07:00
title = "Self-Signed SSL"
+++

I've been working with two-way SSL in Kafka recently and could not find a concise place with the steps of creating your own CA and issuing certificates for the nodes. The same process would apply to SSL for many distributed systems including Cassandra and others. Here's a post with a little background and some scripts to assist.

## What is SSL?
SSL is great as a minimal level of security in any web-based product and something you should always use when available. Personally, I like to use [HTTPS:// Everywhere](https://www.eff.org/https-everywhere) to enforce it is on by default.

SSL or Secure Socket Layer allows for secured communication between two devices, be that your browser and a server or even two servers. In order to do this the hosting server provides a public SSL Certificate which the client uses to encrypt the message before sending it. There is more math and complexity involving cipher suite negotiation and the actual protocol that you can get in to. If you want more on that I suggest checking out [Geek For Geeks - Secure Socket Layer(SSL)](https://www.geeksforgeeks.org/secure-socket-layer-ssl/).

## What is Self Signed SSL?
Once the client server receives the certificate it must then decide if it can trust it or not. Anyone can easily generate certificate with a simple command like
```bash
openssl req -new -newkey rsa:2048 -nodes -x509 -keyout server.key -out server.crt
```
You can even view the results with:
```bash
openssl x509 -noout -text -in server.crt
```
This certificate is known as a 'self signed' certificate. This means that there is no authority that has issued the cert and all the client has to go by is this single certificate.

## Certificate Signing
In order to get a signed certificate, you would generally request a certificate from a trusted Certificate Authority (CA). In turn, that CA might have a certificate signed by another CA. While this would be fun to picture as turtles all the way down, the chain does eventually end. Certificates (at any level) can be trusted manually and placed in a trust store which is nothing more than a repository of trusted certificates. This can be a custom truststore containing certificates that you have chosen to trust or it will be the default truststore containing well known Certificate Authorities known as Root CAs.

## Hostname Verification
The final point before getting into the script is hostname verification. Commonly with self-signed certs, this gets turned off (which you shouldn't do) but using this process you'll be able to define it properly and not need to disable this. On a ceritifcate there is a 'Subject' which looks something like "CN=Hostname,OU=Group,O=Company,C=Country". In this case the CN is the hostname of the server and the client can then compare the hostname to the certificate to verify they match. However, when building internal apps, running a server locally, being behind a vip, or many other reasons they may not match. Instead of disabling this functionality and losing this ecurity, you can simply add a Subject Alternative Name to supply multiple names or even ip addresses.

## Building Your Own CA
Now with distributed systems, we often want encryption between the nodes but these servers don't really need to have certificates owned by externally trusted CAs. It may even be considered safer that they not rely on external CAs to give you full control and thus immediately reject a certificate not issued by you. Simimlarly, an Enterprise may create their own CA (externally signed or not) and preinstall this to your corporate machine's trust store.

For the sake of this project. A simple CA can be made as follows:
```bash
openssl req -new -x509 -nodes -newkey rsa:2048 -subj "/CN=${CA_NAME}" -keyout ${CA_NAME}.key -out ${CA_NAME}.crt
```
There are other solutions for more complex projects but this allows a simple CA creation. From this point on, be very careful with your ca.key file. It is the private key which will be trusted by any client created in this process.

## Generating Client Keystores
This is the step that you will likely do many times. Once you have your new Certificate Authority (CA) you'll need to create some clients. The result will be a java keystore that can be used in any java client. I'll provide a script at the end but first I'll go over the steps.
1. Create Private Key and Keystore
2. Generate CSR (Certificate Signing Request)
3. Create a Signed certificate from your CA
4. Import CA to keystore
4. Import Certificate to keystore
5. Create a new truststore containing only the CA

Here is a genclient script to create new clients from your CA. Called with ./genclient hostname [alt name 1] [alt name n]. I.E. ./genclient server01 localhost server01.my.domain
```bash
#Configure Environement. Please change the passwords
export CA_NAME=${CA_NAME}
export CA_PASSWORD=changeme
export CLIENT_PASSWORD=changeme

#Pull out Client argument. Exit if files exist
CLIENT=$1
shift
if [[ -f ${CLIENT}.keystore ]]; then
  echo "${CLIENT} already exists. Please remove if you wish to create a new certificate."
  exit 1;
fi;

#Confirm CA exists
if [[ ! -f ${CA_NAME}.key ]]; then
  echo "${CA_NAME} does not exist. "
  exit 1;
fi;

#Configure the DName. This can optionally have the full Subject including OU,O,C
export DNAME="CN=${CLIENT}"

#Create SAN List from parameters
export SAN_LIST="DNS:${CLIENT}"
for s in "$@"; do
  SAN_LIST="$SAN_LIST,DNS:$s"
done;

#Gen keystore with private key
keytool -genkeypair -keyalg RSA -alias ${CLIENT} -keystore ${CLIENT}.keystore -storepass $CLIENT_PASSWORD -keypass $CLIENT_PASSWORD -validity 365 -keysize 2048 -dname "$DNAME" -storetype PKCS12 -ext "SAN=$SAN_LIST"

#Gen csr
keytool -keystore ${CLIENT}.keystore -alias ${CLIENT} -certreq -file ${CLIENT}.csr -keypass $CLIENT_PASSWORD -storepass $CLIENT_PASSWORD -dname "$DNAME" -ext "SAN=$SAN_LIST"

#Sign csr with CA
openssl x509 -req -CA ${CA_NAME}.crt -CAkey ${CA_NAME}.key -in ${CLIENT}.csr -out ${CLIENT}.crt -days 365 -CAcreateserial -passin pass:$CLIENT_PASSWORD -extfile <(printf "subjectAltName=$SAN_LIST")

#Import CA to keystore
keytool -keystore ${CLIENT}.keystore -alias ${CA_NAME} -importcert -file ${CA_NAME}.crt -noprompt  -keypass $CLIENT_PASSWORD -storepass $CLIENT_PASSWORD

#Import cert to keystore
keytool -keystore ${CLIENT}.keystore -alias ${CLIENT} -importcert -noprompt -file ${CLIENT}.crt -keypass $CLIENT_PASSWORD -storepass $CLIENT_PASSWORD

#Create truststore
keytool -import -v -trustcacerts -alias ${CA_NAME} -file ${CA_NAME}.crt -keystore ${CLIENT}.truststore -keypass $CLIENT_PASSWORD -storepass $CLIENT_PASSWORD -noprompt
```

## Conclusion
All of these settings can be tweaked and tuned to meet your security requirements. The code in this is a sample to get started and to cover the processes involved. If this is a mission critical system, please consult your internal security team for any applicable policies.
