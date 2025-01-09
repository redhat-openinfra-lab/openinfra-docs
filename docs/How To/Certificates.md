# Certificates

## CA Key and Certificate Files

 The CA .key and .pem files are in the /etc/httpd/conf/ssl.key directory on the DNS Utility server in the same directory.

```
# openssl genrsa -des3 -out openinfraCA.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
.......................................................................................................................................+++++
...........................................................................................+++++
e is 65537 (0x010001)
Enter pass phrase for openinfraCA.key: *******
Verifying - Enter pass phrase for openinfraCA.key: *******


# openssl req -x509 -new -nodes -key openinfraCA.key -sha256 -days 1095 -out openinfraCA.pem
Enter pass phrase for openinfraCA.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:North Carolina
Locality Name (eg, city) [Default City]:Raleigh
Organization Name (eg, company) [Default Company Ltd]:Red Hat
Organizational Unit Name (eg, section) []:OpenInfrastructure Lab
Common Name (eg, your name or your server's hostname) []:openinfra.lab
Email Address []:userName@redhat.com
```

Copy the new CA .pem file to the anchors directory; run update-ca-trust to add it to the list of trusted CA certificates:

```
# cp openinfraCA.pem /usr/share/pki/ca-trust-source/anchors/
# update-ca-trust
```

Use the trust list command to verify the new CA is included in the list of trusted CAs:

```
# trust list | grep -B2 -A2 openinfra.lab 
pkcs11:id=%91%54%10%D3%0D%E4%AD%A7%08%E7%18%EF%A8%62%F7%BF%59%D6%4D%6E;type=cert
    type: certificate
    label: openinfra.lab
    trust: anchor
    category: authority
```    

## Creating a Certificate for a Server/Service

There’s two ways to do this, manual and scripted.  Both are provided here.

### Scripted Process

Login to the Lab DNS server (172.20.129.19).  
Switch to the root user.    
Add an entry in /etc/hosts for the Server/Service you are generating the SSL certificate for.  
Change directory to /root/ssl-certifcates and run the script.  

```
cd /root/ssl-certificates  
./create-certificate.sh <FQDN>  
```

> Example: ./create-certificate jira-sm.openinfra.lab  

You will be prompted for the pass phrase to the openinfraCA.key. 

> NOTE: The pass phrase is in the Cloud Infra Lab spreadsheet.

All output files will start with the FQDN. Example:  

-rw-r--r--. 1 root root 1517 Mar 22 17:16 jira-sm.openinfra.lab.crt  
-rw-r--r--. 1 root root 1054 Mar 22 17:16 jira-sm.openinfra.lab.csr  
-rw-r--r--. 1 root root  254 Mar 22 17:16 jira-sm.openinfra.lab.ext  
-rw-------. 1 root root 1675 Mar 22 17:16 jira-sm.openinfra.lab.key  

### Manual Process

Generate the key file:

```
# openssl genrsa -out cephrgw.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
.........+++++
..............................................................+++++
e is 65537 (0x010001)
```

Generate a Certificate Signing Request file:

```
# openssl req -new -key cephrgw.key -out cephrgw.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:US
State or Province Name (full name) []:North Carolina
Locality Name (eg, city) [Default City]:Raleigh
Organization Name (eg, company) [Default Company Ltd]:Red Hat
Organizational Unit Name (eg, section) []:OpenInfrastructure Lab
Common Name (eg, your name or your server's hostname) []:cephrgw 
Email Address []:youremail@redhat.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:Redhat1!
An optional company name []:      
```

Create an x509 V3 extension config file to define the Subject Alternate Names (SAN)

```
# cat cephrgw.ext
authorityKeyIdentifier = keyid,issuer
basicConstraints = CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = cephrgw.cephlab.openinfra.lab
DNS.2 = cephrgw
DNS.3 = *.cephrgw
DNS.4 = *.cephrgw.cephlab.openinfra.lab
```

Create the signed certificate using the .csr and .ext file along with the openinfraCA.pem and .key files:

```
# openssl x509 -req -in cephrgw.csr -CA openinfraCA.pem -CAkey openinfraCA.key -CAcreateserial -out cephrgw.crt -days 365 -sha256 -extfile cephrgw.ext
Signature ok
subject=C = US, ST = North Carolina, L = Raleigh, O = Red Hat, OU = OpenInfrastructure Lab, CN = cephrgw, emailAddress = bmclaren@redhat.com
Getting CA Private Key
Enter pass phrase for openinfraCA.key: Redhat1!
```


> NOTE: The option –CAcreateserial generates the openinfraCA.srl file.  In subsequent calls to create a signed certificate, use the -CAserial parameter with this file (-CAserial openinfraCA.srl).  The file is used to keep track of the unique serial numbers.

