# Add Self Signed Certificate

  - [Web Security](#Web)
  - [MQTT Security](#MQTT)

## Generate self signed certificates

As many certificates as entities must be generated, e.g. one for each `client` and one for each `server` or `broker`.

### Becoming a CA

To sign your own certificates you need to become a certification authority, for that you need a private key (.key) and a Root Certificate Authority certificate (.pem).

- Generate a key

  ```sh
    openssl genrsa -des3 -out rootCA.key 2048
  ```

- Generate a root certificate

  ```sh
    openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 730 -out rootCA.pem
  ```

- check the root certificate

  ```sh
    openssl x509 -in rootCA.pem -text -noout
  ```

### Create a certificate request

> **Note:** These are the certificates of the entities, i.e. `clients`, `servers` or `brokers`.

- Create a private key to be used during the certificate signing process

  ```sh
    openssl genrsa -out tls.key 2048
  ```

- Use the private key to create a certificate signing request

  ```sh
    openssl req -new -key tls.key -out tls.csr
  ```

- Create a config file openssl.cnf with a list of domain names associated with the certificate (*replace daim.ceit.com with your own domain*)

  ```sh
    # Extensions to add to a certificate request
    basicConstraints       = CA:FALSE
    authorityKeyIdentifier = keyid:always, issuer:always
    keyUsage               = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
    subjectAltName         = @alt_names
    [ alt_names ]
    DNS.1 = daim.ceit.com
  ```

### Sign the certificate request using CA

> **Note:** The following procedure allows signing the certificates of the entities, i.e. `clients`, `servers` or `brokers`.

- To sign the CSR using openssl.cnf and the rootCA created:

  ```sh
    openssl x509 -req \
        -in tls.csr \
        -CA rootCA.pem \
        -CAkey rootCA.key \
        -CAcreateserial \
        -out tls.crt \
        -days 730 \
        -sha256 \
        -extfile openssl.cnf
  ```

- Verify that the certificate is built correctly:

  ```sh
    openssl verify -CAfile rootCA.pem -verify_hostname daim.ceit.com tls.crt
  ```

## Web Security

In this particular case the frontend is using react js with babel, and the backend is using express js.

### React + babel (Frontend)

 we will need to add the --https --key and --cert parameters at the launch command in the package.json file

  ```console
    "start": "babel-node ./node_modules/webpack-dev-server/bin/webpack-dev-server --https --key "route to your key" --cert "route to your cert" --host 0.0.0.0 --open",
  ```

### Express js (Backend)

  The file to change is the one that contains the server startup, In this particular project the file that contains the configuration is backend/bin/www, but it can be other

  ```js
  const https = require('https');
  const fs = require('fs');
  const app = require('../app'); // Recogemos el modulo de app con express
  const port = parseInt(process.env.PORT, 10) || 8001; // Creamos un puerto
  app.set('port', port); // Lo aplicamos
  let options = {
    key: fs.readFileSync('./certs2/tls.key'),
    cert: fs.readFileSync('./certs2/tls.crt')
  };
  //const server = https.createServer(app); // Creamos el servidor
  const server = https.createServer(options, app).listen(port);
  ```

### Install certificate

To use the self signed certificates we will need to tell our computer to trust our ca following this steps:

- windows
    - Go to the windows device certificate manager.
    - Select “Trusted Root Certification Authorities”, right-click “Certificates” and select “All Tasks” then “Import”
    - In the new window select the default options and the rootCA.pem file and save the changes
- Linux (Docker included)
    - Copy the certificate to the /usr/local/share/ca-certificates/ route whit the following command:

    ```sh
    cp rootCA.pem /usr/local/share/ca-certificates/
    ```

## MQTT Security

### Basic Broker Configuration

The broker configuration is done in the file mosquitto.conf

```console
listener 8883 0.0.0.0
cafile /mosquitto/config/rootCA.pem
certfile /mosquitto/config/broker.crt
keyfile /mosquitto/config/broker.key
tls_version tlsv1.2
require_certificate true
persistence false
connection_messages true
allow_anonymous true
```

> **Note:** [Optional] To add user and password based authentication add to the mosquitto.conf file:

```console
password_file /mosquitto/config/passwordfile
```

The password file (passwordfile) contains the user:password combination allowed to use the broker

``` console
CEIT1:$7$101$fHvGO+AtT5AbCJ5y$pRN+N6MhflgtVVTkjsDawWXR4LrCNMcbvNRo3Q3NMcziVmiZKoC8Z94uD1+mffe9VFNg3xGa5sjJUzRYu0YfYQ==
```

The password file is created with the command turn to enter a password for the user

```console
mosquitto_passwd -c passwordfile user
```

To add additional users to the file

```console
mosquitto_passwd -b passwordfile user password
```

### Mosquitto Terminal

Subscriber:

```console
mosquitto_sub -h ip -p port --cafile rootCA.pem  --cert Mymqtt.crt  --key Mymqtt.key  -t "topic" --tls-version tlsv1.2 -u "username" -P "password"
```

Publisher:

```console
mosquitto_pub -h ip -p port --cafile rootCA.pem --cert MyCert.crt --key MyKey.key -t "topic" --tls-version tlsv1.2 -m "message" -u "username" -P "password"
```

### Paho Mqtt Python Client

```console
import paho.mqtt.client as mqtt
import ssl

mqtt_client.username_pw_set(mqtt_username, password=mqtt_password)
mqtt_client.tls_set(ca_certs=CA_PATH, certfile=CERT_PATH, keyfile=KEY_PATH, cert_reqs=ssl.CERT_REQUIRED,tls_version=ssl.PROTOCOL_TLSv1_2, ciphers=None)
mqtt_client.tls_insecure_set(True)
mqtt_client.connect(ip, port=port)
mqtt_client.loop_start()

mqtt_client.subscribe(TOPIC_SUB, qos=0)
```

## Paho Mqtt C/C++ Client

```console
MQTTClient client;
MQTTClient_connectOptions conn_opts = MQTTClient_connectOptions_initializer;
MQTTClient_SSLOptions ssl_opts = MQTTClient_SSLOptions_initializer;
MQTTClient_message pubmsg = MQTTClient_message_initializer;
MQTTClient_deliveryToken delivery_token;

int rc;

string connect_address = "ssl://ip:port";

MQTTClient_create(&client, connect_address.c_str(), CLIENTID, MQTTCLIENT_PERSISTENCE_NONE, NULL);

conn_opts.ssl = &ssl_opts;
conn_opts.ssl->struct_version = 1;
conn_opts.ssl->sslVersion = 3;

conn_opts.ssl->trustStore = "rootCA.pem";
conn_opts.ssl->keyStore = "MyCert.crt.pem";
conn_opts.ssl->privateKey = "MyKey.key.pem";

conn_opts.username = USERNAME;
conn_opts.password = PASSWORD;

conn_opts.keepAliveInterval = 20;
conn_opts.cleansession = 1;

MQTTClient_setCallbacks(client, NULL, connlost, msgarrvd, delivered);
printf("Connecting to ip:port=%s\n", connect_address.c_str());

while ((rc = MQTTClient_connect(client, &conn_opts)) != MQTTCLIENT_SUCCESS)
{
    printf("Failed to connect, return code %d\n", rc);
    printf("New attempt: Connecting to ip:port=%s\n", connect_address.c_str());
}

pubmsg.qos = QOS;
pubmsg.retained = 0;
delivery_token = 0;
```
