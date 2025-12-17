# crt-generation-step-for-kafka-mtls

# Step-by-Step Guide

## 1.Generate CA Key and Self-Signed Certificate (Root Trust Anchor)

```
openssl req -new -x509 -keyout ca-key -out ca-cert -days 3650 -subj "/CN=Kafka-Security-CA" -nodes -passout pass:password
```
- ca-key: Private key (keep secret!).
- ca-cert: Public cert (import into all truststores).

## 2.Generate Broker Key and Certificate Signing Request (CSR) (Repeat for each broker, e.g., kafka1/kafka2/kafka3)
 
- Create config file for SANs (critical for hostname/IP verification):
  Create broker.cnf:

```bash
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = kafka1  # Or broker hostname

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = kafka1
DNS.2 = localhost
IP.1 = 192.168.240.54  # Broker1 IP
IP.2 = 127.0.0.1
```
Adjust for each broker (CN, IPs from advertised.listeners/quorum).

- Generate key + CSR:
```
openssl req -new -newkey rsa:2048 -keyout broker.key -out broker.csr -config broker.cnf -nodes -passout pass:password
```
- Sign with CA:
```
openssl x509 -req -CA ca-cert -CAkey ca-key -in broker.csr -out broker-signed.crt -days 3650 -CAcreateserial -extensions v3_req -extfile broker.cnf -passin pass:password
```

## 3.Create Broker Keystore (PKCS12 then JKS)
```
openssl pkcs12 -export -in broker-signed.crt -inkey broker.key -out broker.p12 -name kafka-broker -passout pass:password -passin pass:password
keytool -importkeystore -srckeystore broker.p12 -destkeystore kafka.keystore.jks -srcstoretype PKCS12 -deststoretype JKS -srcstorepass password -deststorepass password
```
- Rename to kafka.keystore.jks (or per-broker if multi).

## 4.Create Broker Truststore (Import CA)
```
keytool -keystore kafka.truststore.jks -alias CARoot -import -file ca-cert -storepass password -noprompt
```

## 5.Generate Client Key/CSR/Sign (For Producers/Consumers, e.g., Spring Boot/Logstash)
- Config client.cnf:
```
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = client  # Or service name

[v3_req]
extendedKeyUsage = clientAuth  # Important for mTLS client auth
subjectAltName = @alt_names

[alt_names]
DNS.1 = client
```
- Generate + sign (same as broker steps 2-3, rename files to client.key, client.csr, client-signed.crt).

## 6.Create Client Keystore
- Same as broker step 3, rename to kafka.client.keystore.jks.

## 7.Create Client Truststore (Same as Broker Truststore)
- Copy broker truststore or recreate: import CA cert, rename to kafka.client.truststore.jks.

## 8.Credential Files for Confluent Docker (Secure Passwords)
```
echo "password" > keystore_creds
echo "password" > truststore_creds
echo "password" > key_creds  # If separate key password
```
