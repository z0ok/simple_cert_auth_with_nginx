# Nginx + CA_Cert Auth
Several simple steps to create your CA and user certs and config your nginx to use all this stuff.
Don't forget passwords you provide! Write them down to the sticknote and place it under your keyboard ;)

This note is based on [this](https://fardog.io/blog/2017/12/30/client-side-certificate-authentication-with-nginx/). Just a shorter one.

## Important note for MacOS users

If you are going to use this algo for MacOS, you have to generate it with ciphers, that Mac can use. Default algos (used by debians, etc) - are not compatible. Either generate certs and CA on MacOS, or look for algo, that can be used by mac.

Or, generate cert with legacy option:

```
openssl pkcs12 -in user.pfx -nodes -legacy -out user.tmp 
openssl pkcs12 -in user.tmp -export -out mac_user.pfx -legacy
```

## Generating a CA authority and a cert
This step should be done once:  
```
openssl genrsa -des3 -out ca.key 4096
openssl req -new -x509 -days 365 -key ca.key -out ca.crt
```
Copy ca.crt file to your nginx server.

## Generating user cert and pfx
This should be done for every user you'd like to authenticate:  
```
openssl genrsa -des3 -out user.key 4096
openssl req -new -key user.key -out user.csr
openssl x509 -req -days 365 -in user.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out user.crt
openssl pkcs12 -export -out user.pfx -inkey user.key -in user.crt -certfile ca.crt
```
Send the .pfx certs to clients, don't forget about passwords!

## Configuring nginx
Now you can configure your Nginx server for working with certs. Add these to your nginx config file:
To "Server" block:  
```
server {
     listen 443 ssl;
     server_name example.com;
     # Some ssl cert, not connected to this article. Google it)
     ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
     ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

     #Auth block:
     ssl_client_certificate /etc/nginx/client_certs/ca.crt;      # Add this
     # make verification optional, so we can display a 403 message to those  
     # who fail authentication
     ssl_verify_client optional;                                 # Add this
```
To "location" block:  
```
location / {
      if ($ssl_client_verify != SUCCESS) {   # Add this
        return 403;                          # Add this
        #your extra code here                # Add this
      }
```
Done! Restart Nginx and visit https to check if dat works!

UPD: Fun fact: if you set same Organization name for CA and Client certs, Nginx will give you 400 error.
UPD2: Another fun fact: apple devices are soooo modern and secure and dont work with default certs. Use LibreSSL to create certs if you love fruits.
