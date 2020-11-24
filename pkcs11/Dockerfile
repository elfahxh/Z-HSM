FROM registry.access.redhat.com/ubi8/ubi
ARG VERSION=2.3.25

RUN yum install -y wget openssl

RUN mkdir -p /etc/ep11client
RUN chmod -R 777 /etc/ep11client
WORKDIR /etc/ep11client

RUN wget https://github.com/IBM-Cloud/hpcs-pkcs11/releases/download/v${VERSION}/pkcs11-grep11-s390x.so.${VERSION} \
    && wget https://github.com/IBM-Cloud/hpcs-pkcs11/releases/download/v${VERSION}/signing_cert.pem \
    && wget https://github.com/IBM-Cloud/hpcs-pkcs11/releases/download/v${VERSION}/pkcs11-grep11-s390x.so.${VERSION}.sig \
    && wget https://github.com/IBM-Cloud/hpcs-pkcs11/releases/download/v${VERSION}/digicert_cert.pem

RUN openssl x509 -pubkey -noout -in signing_cert.pem -out sigkey.pub \
    && openssl dgst -sha256 -verify sigkey.pub -signature pkcs11-grep11-s390x.so.${VERSION}.sig pkcs11-grep11-s390x.so.${VERSION} \
    && openssl ocsp -no_nonce -issuer digicert_cert.pem -cert signing_cert.pem -VAfile digicert_cert.pem -text -url http://ocsp.digicert.com -respout ocsptest
