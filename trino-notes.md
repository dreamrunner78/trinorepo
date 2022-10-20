# TLS certificates

## CA Authority
```
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 365 -key ca.key -subj "/C=FR/ST=IDF/L=PARIS/O=ABE/OU=ABE, INC./CN=DCP Root CA" -out ca.crt
```
## Server certificate
-addext 'subjectAltName=DNS:hello-bassim.test,DNS:localhost,DNS:tcb-trino,DNS:tcb-trino-coordinator-0' \
```
openssl req -newkey rsa:2048 -nodes -keyout server.key \
    -subj "/C=FR/ST=IDF/L=PARIS/O=ABE/OU=ABE, INC./CN=tcb-trino-coordinator-0.trino-headless.trino.svc.cluster.local" -out server.csr

cat << EOF > localhost.ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
subjectAltName = @alt_names
[alt_names]
DNS.1 = hello-bassim.test
DNS.2 = localhost
DNS.3 = tcb-trino
DNS.4 = tcb-trino-coordinator-0
EOF

openssl x509 -req -extfile /tmp/localhost.ext \
    -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt
```

## Generate keystore & truststore
```
openssl pkcs12 -export -out server.p12 -name "trino" -inkey server.key -in server.crt -passout pass:"bassim"
keytool -importkeystore -srckeystore server.p12 -srcstoretype PKCS12 -srcstorepass "bassim" -deststorepass "bassim" -destkeystore server.jks

keytool -import -file "ca.crt" \
    -keystore "truststore.jks" \
    -storepass "bassim" \
    -noprompt
```


## Kubernetes secret
```
kubectl -n trino create secret generic trino-tls-secret --from-file=tls.crt=server.crt --from-file=tls.key=server.key --from-file=ca.crt

kubectl -n trino create secret generic trino-tls-secret --from-file=tls.crt=ca.crt --from-file=tls.key=ca.key --from-file=ca.crt
```