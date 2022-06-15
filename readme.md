# Add tls encryption to js frontend/backend

## Enviroment

In this particular case the frontend is using react js with babel, and the backend is using expres js.

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

- Create a config file openssl.cnf with a list of domain names associated with the certificate (replace daim.ceit.com with your hown domain)

> basicConstraints       = CA:FALSE
> authorityKeyIdentifier = keyid:always, issuer:always
> keyUsage               = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
> subjectAltName         = @alt_names
> [ alt_names ]
> DNS.1 = daim.ceit.com

### Sign the certificate request using CA

- To sign the CSR using openssl.cnf:

  ``` sh
    openssl genrsa -des3 -out rootCA.key 2048
    ```

- Generate a root certificate

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

### React + babel

in the launch command we will need to add the --https parameter at the launch command in the packaje.json file

``` sh
    "start": "babel-node ./node_modules/webpack-dev-server/bin/webpack-dev-server --https --key  -- --host 0.0.0.0 --open",
  ```

To use the self signed certificates we will need to tell our computer to trust our ca following this steps:

- windows

- Linux
