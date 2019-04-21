# CA

```bash
$ openssl genrsa -out rootCA.key 2048

$ openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=*.ca-domain.com" -days 5000 -out rootCA.crt
```

# Server

```bash
$ openssl genrsa -out server.key 2048

$ openssl req -new -key server.key -subj "/CN=*.server-domain.com" -out server.csr

$ openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 5000
```

# Client

```bash
$ openssl genrsa -out client.key 2048

$ openssl req -new -key client.key -subj "/CN=client-id" -out client.csr

$ echo "extendedKeyUsage=clientAuth" > client.ext

$ openssl x509 -req -in client.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -extfile client.ext -out client.crt -days 5000
```
