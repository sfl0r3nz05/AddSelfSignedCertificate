# Add tls encryption to js frontend/backend

## Environment

In this particular case the frontend is using react js with babel, and the backend is using express js.

## Generate certificates

### Becoming a ca

to sign your own certificates you need to become a certification authority, for that you need a private key (.key) and a Root Certificate Authority certificate (.pem).

- Generate a key

  ``` sh
    openssl genrsa -des3 -out rootCA.key 2048
    ```

- Generate a root certificate

  ``` sh
   openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 730 -out rootCA.pem
    ```

- check the root certificate

  ``` sh
   openssl x509 -in rootCA.pem -text -noout
    ```

### Create a certificate request

- Create a private key to be used during the certificate signing process

  ``` sh
    openssl genrsa -out tls.key 2048
    ```

- Use the private key to create a certificate signing request

  ``` sh
   openssl req -new -key tls.key -out tls.csr
    ```

- Create a config file openssl.cnf with a list of domain names associated with the certificate (replace daim.ceit.com with your own domain)
  ``` sh
  # Extensions to add to a certificate request
  basicConstraints       = CA:FALSE
  authorityKeyIdentifier = keyid:always, issuer:always
  keyUsage               = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
  subjectAltName         = @alt_names
  [ alt_names ]
  DNS.1 = daim.ceit.com
    ```

### Sign the certificate request using CA

- To sign the CSR using openssl.cnf and the rootCA created:

  ``` sh
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

- verify that the certificate is built correctly:

  ``` sh
    openssl verify -CAfile rootCA.pem -verify_hostname daim.ceit.com tls.crt
    ```

## Implementation

### React + babel (Frontend)

 we will need to add the --https --key and --cert parameters at the launch command in the package.json file

``` sh
    "start": "babel-node ./node_modules/webpack-dev-server/bin/webpack-dev-server --https --key "route to your key" --cert "route to your cert" --host 0.0.0.0 --open",
  ```
### Express js (Backend)

  The file to change is the one that contains the server startup, In this particular project the file that contains the configuration is backend/bin/www, but it can be other

``` js  
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
    ``` sh
    cp rootCA.pem /usr/local/share/ca-certificates/
  ```
