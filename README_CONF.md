# Kafka SSL Setup Guide for Beginners

This guide walks you through setting up SSL (Secure Sockets Layer) on a Kafka cluster. SSL ensures that communication between Kafka brokers and clients is encrypted and secure.

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Step 1: Create a Certificate Authority (CA)](#step-1-create-a-certificate-authority-ca)
4. [Step 2: Create a Truststore and Import the CA](#step-2-create-a-truststore-and-import-the-ca)
5. [Step 3: Create a Keystore for Each Kafka Broker](#step-3-create-a-keystore-for-each-kafka-broker)
6. [Step 4: Create a Certificate Signing Request (CSR)](#step-4-create-a-certificate-signing-request-csr)
7. [Step 5: Sign the CSR Using the CA](#step-5-sign-the-csr-using-the-ca)
8. [Step 6: Import the CA Certificate into the Broker Keystore](#step-6-import-the-ca-certificate-into-the-broker-keystore)
9. [Step 7: Import the Signed Certificate into the Broker Keystore](#step-7-import-the-signed-certificate-into-the-broker-keystore)
10. [Step 8: Configure Kafka for SSL](#step-8-configure-kafka-for-ssl)
11. [Step 9: Start or Restart Kafka Brokers](#step-9-start-or-restart-kafka-brokers)
12. [Troubleshooting](#troubleshooting)

---

## Introduction

Apache Kafka supports SSL to secure communication between brokers and between brokers and clients. SSL ensures that data in transit is encrypted and authenticated. This guide will show you how to configure SSL in Kafka from scratch.

---

## Prerequisites

Before you start, ensure that you have the following tools installed:
- **OpenSSL**: Used to create keys and certificates.
- **Java Keytool**: Comes with the JDK, used to create keystores and truststores.
- **Kafka**: A working Kafka setup (standalone or Docker-based).

---

## Step 1: Create a Certificate Authority (CA)

The Certificate Authority (CA) signs the certificates for Kafka brokers. This makes the CA the root of trust in your SSL setup.

### Command:
```bash
openssl req -new -x509 -days 365 -keyout ca-key -out ca-cert -subj "/C=DE/ST=NRW/L=MS/O=YourOrg/OU=Kafka/CN=Root-CA" -passout pass:kafka-broker


Explanation:
-new -x509: Creates a new certificate.
-days 365: The certificate will be valid for 1 year.
ca-key: The private key for your CA.
ca-cert: The public CA certificate, which will sign broker certificates.
The subject (-subj) defines your certificate details (Country, State, Organization, etc.).


## Step 2: Create a Truststore and Import the CA
Each Kafka broker needs to trust the CA. To do this, we create a truststore, which is a place to store trusted certificates like the CA's public certificate.


keytool -keystore kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt


Explanation:
kafka.server.truststore.jks: The truststore file that will store the CA certificate.
-storepass: Password for the truststore.
-import: We are importing the CA certificate.
-alias ca-root: The alias used to identify the CA certificate.
-file ca-cert: The CA certificate file created in Step 1.


Step 3: Create a Keystore for Each Kafka Broker
Each Kafka broker needs its own keystore, which holds its private and public keys. This step generates a keypair (private and public key) for each broker.

Command (for broker kafka1):

keytool -keystore kafka.server.keystore.jks -storepass kafka-broker -alias kafka1 -validity 365 -keyalg RSA -genkeypair -keypass kafka-broker -dname "CN=kafka1,OU=Kafka,O=YourOrg,L=City,ST=State,C=DE"


Explanation:
-keystore kafka.server.keystore.jks: The keystore file that will store the broker’s keys.
-alias kafka1: Alias for this broker’s certificate.
-genkeypair: Generates a public-private keypair.
-dname: The Distinguished Name (DN) used for this broker’s certificate (common name CN=kafka1).
Repeat this step for each broker (kafka2, kafka3, etc.), changing the alias and CN accordingly.


Step 4: Create a Certificate Signing Request (CSR)
A CSR is needed to get the broker’s certificate signed by the CA.

Command (for broker kafka1):

keytool -alias kafka1 -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass kafka-broker -keypass kafka-broker


Explanation:
-certreq: Creates a CSR (certificate signing request).
-file cert-file: The CSR output file (to be signed by the CA).
Repeat this for each broker.


Step 5: Sign the CSR Using the CA
Now, the CA signs the CSR to issue the broker certificate.

Command (for broker kafka1):


openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:kafka1,DNS:localhost")


Explanation:
-req: Sign the CSR using the CA.
-CA ca-cert: The CA certificate.
-CAkey ca-key: The CA private key.
cert-file: The CSR file generated in Step 4.
cert-signed: The signed certificate file output.
-extensions SAN: Adds Subject Alternative Names (SAN) like DNS entries.
Repeat this step for each broker, adjusting the SAN as needed.


Step 6: Import the CA Certificate into the Broker Keystore
Each broker’s keystore needs to trust the CA before it can import its signed certificate.

Command (for broker kafka1):


keytool -importcert -keystore kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt


Explanation:
-importcert: Import the CA certificate into the broker’s keystore.
-alias ca-root: Use the CA alias.
Repeat for each broker.


Step 7: Import the Signed Certificate into the Broker Keystore
Now, import the signed certificate for each broker.

Command (for broker kafka1):


keytool -keystore kafka.server.keystore.jks -alias kafka1 -import -file cert-signed -storepass kafka-broker -keypass kafka-broker -noprompt


Explanation:
-import: Import the signed certificate into the broker’s keystore.
cert-signed: The signed certificate generated in Step 5.
Repeat for each broker.


Step 8: Configure Kafka for SSL
Now, we need to configure each Kafka broker to use SSL by modifying the server.properties file.

Add the following properties to each broker’s server.properties file:


listeners=SSL://:9093
ssl.keystore.location=/path/to/kafka.server.keystore.jks
ssl.keystore.password=kafka-broker
ssl.key.password=kafka-broker
ssl.truststore.location=/path/to/kafka.server.truststore.jks
ssl.truststore.password=kafka-broker
ssl.endpoint.identification.algorithm=
ssl.client.auth=required


Explanation:
listeners=SSL://:9093: Sets the broker to listen on port 9093 with SSL.
ssl.keystore.location: Path to the broker’s keystore.
ssl.truststore.location: Path to the truststore with the CA certificate.
ssl.client.auth=required: Ensures mutual authentication between brokers.
Repeat for each broker, adjusting the listener port (9094, 9095, etc.).

Step 9: Start or Restart Kafka Brokers
If you're using Docker Compose, restart your Kafka containers:

docker-compose restart


If running Kafka manually, restart each broker using your standard method (e.g., systemctl or bin/kafka-server-start.sh).

Troubleshooting
Common Issues:
Incorrect passwords: Ensure the passwords for the keystore and truststore in server.properties match those used during setup.
Certificate errors: If the broker cannot validate a certificate, check if the correct CA and signed certificates are being used.
Port conflicts: Ensure each broker listens on a unique port (e.g., 9093, 9094, 9095).
