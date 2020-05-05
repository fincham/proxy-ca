# proxy-ca

Scripts to generate single-use Certificate Authorities for proxy <-> backend comms.

## Usage

For an app called "foo", run `make-bundle foo`. This will generate these files:

```
fincham@lidar:~/tmp$ /opt/hotplate/proxy-ca/make-bundle foo
*** Generating new bundle for 'foo'...
*** Setting up the CA...
[...]
*** Signing CSRs...
Signature ok
subject=CN = foo-proxy
Getting CA Private Key
Signature ok
subject=CN = foo-app
Getting CA Private Key
*** Cleaning up CA...
*** The cert and key are in 'foo-{proxy,app}-{cert,key}.pem'
*** The proxy combined cert/key bundle is in 'foo-proxy-combined.pem'
fincham@lidar:~/tmp$ ls -l
total 36
-rw------- 1 fincham fincham 1655 Dec  5 13:04 foo-app-cert.pem
-rw------- 1 fincham fincham 1582 Dec  5 13:04 foo-app-csr.pem
-rw------- 1 fincham fincham 3243 Dec  5 13:04 foo-app-key.pem
-rw------- 1 fincham fincham 1793 Dec  5 13:04 foo-ca-cert.pem
-rw------- 1 fincham fincham 1655 Dec  5 13:04 foo-proxy-cert.pem
-rw------- 1 fincham fincham 4898 Dec  5 13:04 foo-proxy-combined.pem
-rw------- 1 fincham fincham 1586 Dec  5 13:04 foo-proxy-csr.pem
-rw------- 1 fincham fincham 3243 Dec  5 13:04 foo-proxy-key.pem
```

You can then run `gimme-app foo` to get a copy/pastable script which will re-generate the necessary keys and certificates on the app server. Similarly for `gimme-proxy foo`.

```
fincham@lidar:~/tmp$ /opt/hotplate/proxy-ca/gimme-app foo
touch foo-app-key.pem
chmod 0600 foo-app-key.pem
cat > foo-app-cert.pem <<EOF
-----BEGIN CERTIFICATE-----
MIIEmDCCAoACAhABMA0GCSqGSIb3DQEBCwUAMBExDzANBgNVBAMMBmZvbyBDQTAe
[...]
+e6I/1UDoVFXM0J1PrhqoNF6Owq/vrJlhrxaQA==
-----END CERTIFICATE-----
EOF
cat > foo-app-key.pem <<EOF
-----BEGIN RSA PRIVATE KEY-----
MIIJKAIBAAKCAgEAxAEbuzTMc4GFnCD19H0fMW9ee51l7yZUHhSZu5ct8rhmzwdi
[...]
GDg58sVI9KzjCJ+YdQHBcux3bedV6R2bb6GeFHpnJX+T4lwa51xb01GbmMo=
-----END RSA PRIVATE KEY-----
EOF
cat > foo-ca-cert.pem <<EOF
-----BEGIN CERTIFICATE-----
MIIFADCCAuigAwIBAgIUG5t++2ez/GqHvHegsg85+7mkxdUwDQYJKoZIhvcNAQEL
[...]
8ybGiuPdC9xd/KZAtdI0pobJLcgpBsMJqu7U0RvNYPoWH3GM
-----END CERTIFICATE-----
EOF
```

## Bugs

There is not a lot of error checking so e.g. overwriting existing files is easily possible.

## Example Apache configurations

### Backend / application

SSLEngine on

SSLProtocol TLSv1.2
SSLCipherSuite ECDHE-RSA-AES256-GCM-SHA384
SSLHonorCipherOrder on
SSLCompression off

SSLCertificateFile /srv/tls/foo-app-cert.pem
SSLCertificateKeyFile /srv/tls/foo-app-key.pem

SSLCACertificateFile /srv/tls/foo-ca-cert.pem
SSLVerifyClient require
SSLVerifyDepth 1

# The proxy will always have serial 0x1000, this prevents 
# other backends from connecting using their own certs.
<If "%{SSL_CLIENT_M_SERIAL} != 1000">
	Require all denied
</If>

### Frontend / reverse proxy

# to (hypothetically) avoid HTTP desync attacks
SetEnv proxy-nokeepalive 1

SSLProxyEngine on
SSLProxyVerify on
SSLProxyCheckPeerName off

ProxyPass / https://foo-app.example.com/
ProxyPassReverse / https://foo-app.example.com/
UseCanonicalName on
ProxyPreserveHost on

SSLProxyCACertificateFile /srv/tls/foo/foo-ca-cert.pem
SSLProxyMachineCertificateFile /srv/tls/foo/foo-proxy-combined.pem
SSLCertificateFile /etc/letsencrypt/live/foo.example.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/foo.example.com/privkey.pem
