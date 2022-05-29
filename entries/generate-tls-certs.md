# Generate self-signed TLS/SSL certificates in seconds  
Parameter explanation coming soon (sometime)


Collected from different sources.  
Wiki link to [x.509](https://en.wikipedia.org/wiki/X.509) for reference.  
`man openssl`, `man x509v3_config` for command reference.  

- [Generating certificates](#generate-a-self-signed-certificate-for-quick-testing)  
  - [Certificate with defaults](#to-create-a-new-x509-certificate-with-all-defaults)  
  - [Certificate with defaults and extension](#to-create-a-new-x509-certificate-with-all-defaults-and-some-extensions)  
  - [Certificate with extensions and details](#to-create-a-new-x509-certificate-with-extensions-and-information-about-certificate-holder)    

## Generate a self-signed certificate for quick testing  
* You need the latest openssl for some options to work.  
* When using the `-newkey` option, a password is "required" when generating the new key.   

### To create a new x.509 certificate with all defaults  

    openssl req -x509 -new -newkey rsa:2048 -out cert.pem -keyout key.pem  

To view the certificate  

    openssl x509 -in cert.pem -noout -text

### To create a new x.509 certificate with all defaults and some extensions
    
    openssl req -x509 -new -newkey rsa:4096 -out cert.pem -keyout key.pem -addext keyUsage=digitalSignature,keyEncipherment  

To view the certificate  

    openssl x509 -in cert.pem -noout -text  

To view only the extensions in this certificate, specifically keyUsage  
    
    openssl x509 -in cert.pem -noout -ext keyUsage

### To create a new x.509 certificate with extensions and information about certificate holder

    openssl req -x509 \
                -new \
                -newkey rsa:2048 \
                -out cert.pem \
                -keyout key.pem \
                -addext  keyUsage=digitalSignature,keyEncipherment \
                -subj "/CN=c3JpbmkK/C=DE/ST=Hessen/L=FFM"

To view only the subject information from this certificate  

    openssl x509 -in cert.pem -noout -subject
    
### Using a pre-generated key

    openssl genrsa -out keyIn.pem 2048
    openssl req -x509 -new -key keyIn.pem -out cert.pem
    
Since the key is generated before generating the certificate, there is no prompt for password this time, therefore we can completely automate the certificate generation step. Putting it all together

    openssl genrsa -out keyIn.pem 2048
    openssl req -new \
                -x509 \
                -key keyIn.pem \
                -out cert.pem \
                -keyout key.pem \
                -addext  keyUsage=digitalSignature,keyEncipherment \
                -subj "/CN=c3JpbmkK/C=DE/ST=Hessen/L=FFM"
    echo
    openssl x509 -in cert.pem -noout -text
    openssl x509 -in cert.pem -noout -ext keyUsage
    openssl x509 -in cert.pem -noout -subject
    

