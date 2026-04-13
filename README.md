## Architecture Overview

The system uses a Certificate Authority (CA) to establish trust between the MQTT broker and client devices.
Mutual TLS (mTLS) ensures both sides authenticate each other before communication begins.

###  Explanation

 **Certificate Authority (CA)**: Issues and signs certificates for both broker and clients  
 **MQTT Broker (Mosquitto)**: Acts as the secure message hub on port 8883  
 **Client Devices**: Sensors or applications authenticated using client certificates  
 **mTLS Authentication**: Both client and server verify each other before connection  
 **Wireshark Verification**: TLS 1.3 handshake and encrypted traffic confirmed at packet level  

This ensures:
 End to end encryption  
 Strong device authentication  
 Unauthorized access prevention  

Three security properties delivered:

  Encryption  TLS 1.3 all data encrypted in transit  Wireshark only "Application Data" visible, no plaintext
  AuthenticationBoth broker and client present certificates Client Hello (SNI=localhost) + Server Hello in capture
  TrustCA signs all certs  no self signed bypass cafile ca.crt verified on connection



 Prerequisites
bash# Ubuntu / Debian
sudo apt update
sudo apt install mosquitto mosquitto-clients openssl wireshark -y
ToolVersion UsedPurposeMosquitto2.xMQTT BrokerOpenSSL3.xCertificate generationWireshark4.xNetwork verificationUbuntu22.04 LTSHost OS

 Step-by-Step Setup
Step 1 Create the Certificate Authority (CA)
bashmkdir mqtt_certs && cd mqtt_certs

# Generate CA private key
openssl genrsa -out ca.key 2048

# Generate CA self signed certificate (valid 10 years)
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/C=IN/ST=TamilNadu/L=Salem/O=IIoT-Lab/CN=MyCA"
  
Step 2 Create the Broker Certificate
bash# Broker private key
openssl genrsa -out broker.key 2048

# Broker certificate signing request
openssl req -new -key broker.key -out broker.csr \
  -subj "/C=IN/ST=TamilNadu/L=Salem/O=IIoT-Lab/CN=localhost"

# Sign broker cert with CA
openssl x509 -req -days 3650 -in broker.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out broker.crt
  
Step 3 Create the Client Certificate
bash# Client private key
openssl genrsa -out client.key 2048

# Client CSR
openssl req -new -key client.key -out client.csr \
  -subj "/C=IN/ST=TamilNadu/L=Salem/O=IIoT-Lab/CN=sensor-client-01"

# Sign client cert with CA
openssl x509 -req -days 3650 -in client.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client.crt
  
Your mqtt_certs/ folder should now contain:

mqtt_certs/
├── ca.crt          ← Certificate Authority (shared with all)
├── ca.key          ← CA private key (NEVER share this)
├── broker.crt      ← Broker certificate
├── broker.key      ← Broker private key
├── client.crt      ← Client certificate
└── client.key      ← Client private key

Step 4 Configure Mosquitto for mTLS
Create /etc/mosquitto/conf.d/mtls.conf:
conf# Standard MQTT port (disable for security-only setup)
# listener 1883

# Secure MQTT port with mTLS
listener 8883

# TLS certificates
cafile   /home/YOUR_USER/mqtt_certs/ca.crt
certfile /home/YOUR_USER/mqtt_certs/broker.crt
keyfile  /home/YOUR_USER/mqtt_certs/broker.key

# Force client certificate verification (THIS is what makes it mTLS)
require_certificate true

# TLS version
tls_version tlsv1.3

# Logging
log_type all
bash# Restart Mosquitto
sudo systemctl restart mosquitto
sudo systemctl status mosquitto

Step 5 Subscribe (Terminal 1)
bashmosquitto_sub -h localhost -p 8883 \
  --cafile ~/mqtt_certs/ca.crt \
  --cert ~/mqtt_certs/client.crt \
  --key ~/mqtt_certs/client.key \
  -t test -v
Step 6 — Publish (Terminal 2)
bashcd ~/mqtt_certs

mosquitto_pub -h localhost -p 8883 \
  --cafile ~/mqtt_certs/ca.crt \
  --cert ~/mqtt_certs/client.crt \
  --key ~/mqtt_certs/client.key \
  -t test -m "hello secure"
Expected subscriber output:

Client null sending SUBSCRIBE (Mid: 1, Topic: test, QoS: 0)
Client null received SUBACK
Client null received PUBLISH (d0, q0, r0, m0, 'test', ... (12 bytes))
hello secure

Step 7 Verify in Wireshark
bash# Open Wireshark
wireshark &

# Apply display filter to see only broker traffic

# In filter bar type:
tcp.port == 8883

What you should see:
TLSv1.3 Client Hello with SNI=localhost
TLSv1.3 Server Hello + Change Cipher Spec
TLSv1.3 Application Data (all MQTT payload encrypted, unreadable)
NO plaintext MQTT PUBLISH frames proof of encryption


References & Standards

MQTT v3.1.1 Specification — OASIS
Eclipse Mosquitto Documentation
IEC 62443 Industrial Cybersecurity Standard
RFC 8446 TLS 1.3
paho mqtt Python Client

If you are working on IIoT, industrial automation, or predictive maintenance let's connect.
Surendar S IIoT 
linkedin.com/in/surendar-srinivasan26
Star this repo if it helped you understand mTLS for industrial applications.
