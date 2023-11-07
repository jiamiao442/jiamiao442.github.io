# 生产中harbor从http升级到https
uuid: 3287bdfe-52d5-3ced-f552-f2589ee030c4
uuid: 43c64337-7010-a3f2-5e63-f96955db17ed

##  Generate a Certificate Authority Certificate

In a production environment, you should obtain a certificate from a CA. In a test or development environment, you can generate your own CA. To generate a CA certficate, run the following commands.

1. Generate a CA certificate private key.

   ```sh
   mkdir /opt/cert
   cd /opt/cert
   openssl genrsa -out ca.key 4096
   ```

2. Generate the CA certificate.

   Adapt the values in the `-subj` option to reflect your organization. If you use an FQDN to connect your Harbor host, you must specify it as the common name (`CN`) attribute.

   ```sh
   openssl req -x509 -new -nodes -sha512 -days 3650 \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=10.165.6.63" \
    -key ca.key \
    -out ca.crt
   ```

## Generate a Server Certificate

The certificate usually contains a `.crt` file and a `.key` file, for example, `10.165.6.63.crt` and `10.165.6.63.key`.

1. Generate a private key.

   ```sh
   openssl genrsa -out 10.165.6.63.key 4096
   ```

2. Generate a certificate signing request (CSR).

   Adapt the values in the `-subj` option to reflect your organization. If you use an FQDN to connect your Harbor host, you must specify it as the common name (`CN`) attribute and use it in the key and CSR filenames.

   ```sh
   openssl req -sha512 -new \
       -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=10.165.6.63" \
       -key 10.165.6.63.key \
       -out 10.165.6.63.csr
   ```

3. Generate an x509 v3 extension file.

   Regardless of whether you’re using either an FQDN or an IP address to connect to your Harbor host, you must create this file so that you can generate a certificate for your Harbor host that complies with the Subject Alternative Name (SAN) and x509 v3 extension requirements. Replace the `DNS` entries to reflect your domain.

   ```sh
   cat > v3.ext <<-EOF
   authorityKeyIdentifier=keyid,issuer
   basicConstraints=CA:FALSE
   keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
   extendedKeyUsage = serverAuth
   subjectAltName = @alt_names
   
   [alt_names]
   IP.1=10.165.6.63
   EOF
   ```

     重点是[alt_names]，这里写的ip地址是最后认证的，比较重要。端口不需要，一旦认证了ip以后所有端口都可以是https的

   [alt_names]： 后面为备用名称列表，可以是域名、泛域名、IP地址

4. Use the `v3.ext` file to generate a certificate for your Harbor host.

   Replace the `10.165.6.63` in the CRS and CRT file names with the Harbor host name.

   ```sh
   openssl x509 -req -sha512 -days 3650 \
       -extfile v3.ext \
       -CA ca.crt -CAkey ca.key -CAcreateserial \
       -in 10.165.6.63.csr \
       -out 10.165.6.63.crt
   ```

## Provide the Certificates to Harbor and Docker

After generating the `ca.crt`, `10.165.6.63.crt`, and `10.165.6.63.key` files, you must provide them to Harbor and to Docker, and reconfigure Harbor to use them.

1. Copy the server certificate and key into the certficates folder on your Harbor host.

   ```sh
   cp 10.165.6.63.crt /data/cert/
   cp 10.165.6.63.key /data/cert/
   ```

2. Convert `10.165.6.63.crt` to `10.165.6.63.cert`, for use by Docker.

   The Docker daemon interprets `.crt` files as CA certificates and `.cert` files as client certificates.

   ```sh
   openssl x509 -inform PEM -in 10.165.6.63.crt -out 10.165.6.63.cert
   ```

3. Copy the server certificate, key and CA files into the Docker certificates folder on the Harbor host. You must create the appropriate folders first.

   ```sh
   cp 10.165.6.63.cert /etc/docker/certs.d/10.165.6.63/
   cp 10.165.6.63.key /etc/docker/certs.d/10.165.6.63/
   cp ca.crt /etc/docker/certs.d/10.165.6.63/
   ```

   If you mapped the default `nginx` port 443 to a different port, create the folder `/etc/docker/certs.d/10.165.6.63:port`, or `/etc/docker/certs.d/harbor_IP:port`.

4. Restart Docker Engine.

   ```sh
   systemctl restart docker
   ```

You might also need to trust the certificate at the OS level. See [Troubleshooting Harbor Installation](https://goharbor.io/docs/2.0.0/install-config/troubleshoot-installation/#https) for more information.

The following example illustrates a configuration that uses custom certificates.

```fallback
/etc/docker/certs.d/
    └── 10.165.6.63:5000
       ├── 10.165.6.63.cert  <-- Server certificate signed by CA
       ├── 10.165.6.63.key   <-- Server key signed by CA
       └── ca.crt               <-- Certificate authority that signed the registry certificate
```

## Deploy or Reconfigure Harbor

If you have not yet deployed Harbor, see [Configure the Harbor YML File](https://goharbor.io/docs/2.0.0/install-config/configure-yml-file/) for information about how to configure Harbor to use the certificates by specifying the `hostname` and `https` attributes in `harbor.yml`.

If you already deployed Harbor with HTTP and want to reconfigure it to use HTTPS, perform the following steps.

1. Run the `prepare` script to enable HTTPS.

   Harbor uses an `nginx` instance as a reverse proxy for all services. You use the `prepare` script to configure `nginx` to use HTTPS. The `prepare` is in the Harbor installer bundle, at the same level as the `install.sh` script.

   ```sh
   ./prepare
   ```

2. If Harbor is running, stop and remove the existing instance.

   Your image data remains in the file system, so no data is lost.

   ```sh
   docker-compose down -v
   ```

3. Restart Harbor:

   ```sh
   docker-compose up -d
   ```

## Verify the HTTPS Connection

After setting up HTTPS for Harbor, you can verify the HTTPS connection by performing the following steps.

- Open a browser and enter [https://10.165.6.63](https://10.165.6.63/). It should display the Harbor interface.

  Some browsers might show a warning stating that the Certificate Authority (CA) is unknown. This happens when using a self-signed CA that is not from a trusted third-party CA. You can import the CA to the browser to remove the warning.

- On a machine that runs the Docker daemon, check the `/etc/docker/daemon.json` file to make sure that the `-insecure-registry` option is not set for [https://10.165.6.63](https://10.165.6.63/).

- Log into Harbor from the Docker client.

  ```sh
  docker login 10.165.6.63:5000
  ```

  If you’ve mapped `nginx` 443 port to a different port,add the port in the `login` command.

  ```sh
  docker login 10.165.6.63:5000
  ```

## 

升级完成。