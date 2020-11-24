

Before this guide, [Configuring a node to use a Hardware Security Module (HSM)](https://test.cloud.ibm.com/docs/blockchain-sw-251?topic=blockchain-sw-251-ibp-console-adv-deployment#ibp-console-adv-deployment-cfg-hsm) can be refereced

## Build an HSM client image on a linuxOne server

Upload the [Dockerfile](./Dockerfile) to a linuxOne server

Build the docker image, and then push the docker image to the docker registry

```
export HPCS_PKCS11_CLIENT_VERSION=<<Release version of PKCS11 client library>>
export HSM_CLIENT_IMAGE_URL=<<The generated docker image url>>

docker build --no-cache --build-arg VERSION=${HPCS_PKCS11_CLIENT_VERSION} -t ${HSM_CLIENT_IMAGE_URL} -f Dockerfile .
docker push ${HSM_CLIENT_IMAGE_URL}
```

For example

```
docker build --no-cache --build-arg VERSION=2.3.29 -t hexiaohu/pkcs11-ubi:s390x-1.0.1 -f Dockerfile .
docker push hexiaohu/pkcs11-ubi:s390x-1.0.1
```

## Create PKCS instance and generate api keys

To follow this guide of PKCS [Best practices for setting up PKCS #11 user types](https://cloud.ibm.com/docs/hs-crypto?topic=hs-crypto-best-practice-pkcs11-access)

## Generate PKCS11 Client Library configuration file

To follow this guide [Performing cryptographic operations with the PKCS #11 API](https://cloud.ibm.com/docs/hs-crypto?topic=hs-crypto-set-up-pkcs-api#step3-setup-configuration-file) to generate client library configuration file, [grep11client.yaml sample](./samples/grep11client.yaml) can be referenced.

The configuration file format as below:

```
amcredentialtemplate: &defaultiamcredential
          enabled: true
          endpoint: "https://iam.test.cloud.ibm.com"
          #keep apikey empty since it is overriden by user configure
          apikey:
          #All users have same instance ID
          instance: <<instanceID>>
tokens:
  0:
    grep11connection:
      address: ep11.us-south.hs-crypto.test.cloud.ibm.com
      port: 9846
      tls:
        enabled: true
        mutual: false
        cacert:
        certfile:
        keyfile:
    storage:
      filestore:
        enabled: false
        storagepath:
      remotestore:
        enabled: true
      # localpostgres:
      #   enabled: false
      #   connectionstring:
    users:
      0: # SO User
        name: "SO user"
        iamauth:
          <<: *defaultiamcredential
      1: # User
        name: "Normal user"
        tokenspaceID: "<<tokenspaceID>>"
        iamauth:
          <<: *defaultiamcredential
      2: # Anonymous user
        name: "Anonymous"
        tokenspaceID: "<<tokenspaceID>>"
        iamauth:
          <<: *defaultiamcredential
          # this API key must exist to overide apikey in defaultcredentials.iamauth.apikey
          apikey: <<APIKEY_OF_ANONYMOUS_USER>
logging:
  loglevel: warning
  logpath: /tmp/pkcs11client.txt
```
   
## Execute pkcs11-tool to initialize the slot

Before the configuration is configured by IBP, the slot must be initialized out of the box

To copy the generated config file, grep11client.yaml, to a local folder /etc/ep11client, and download the pkcs11 client [.so](https://github.com/IBM-Cloud/hpcs-pkcs11/releases), amd64 or s390x both are ok, into the local folder /etc/ep11client also, and then use pkcs11-tool (Any other tool is also ok) to initilize the slot

The api-key (as the PIN) of the SO user must be provided

```
export PKCS11_SO_PIN=STRING_FOR_API_KEY_OF_SO_USER
export PKCS11_LABEL=LABEL_OF_THE_SLOT

pkcs11-tool --module=/etc/ep11client/pkcs11-grep11-s390x.so --init-token --so-pin ${PKCS11_SO_PIN}  --label ${LABEL_OF_THE_SLOT}
```

## Create secret and configmap in namespace

To create a secret to carry content of PKCS11 Client Library configuration file, grep11client.yaml

```
export NAMESPACE=<<NAMESPACE>>
kubectl create secret generic hsmcrypto --from-file=grep11client.yaml -n ${NAMESPACE}
```

To create a configmap be used by IBP init container, a [ibp-hsm-config.yaml sample](./samples/ibp-hsm-config.yaml) can be referenced

The configmap format as below

```
apiVersion: v1
data:
  ibp-hsm-config.yaml: |
    envs:
    - name: EP11CLIENT_CFG
      value: <<PATH to the mount configration file>>
    mountpaths:
    - name: hsmconfig
      secret: hsmcrypto
      mountpath: <<PATH to the configration file>>
      subpath: grep11client.yaml
    library:
      filepath: <<PATH to the PKCS11 client library file>>
      image: <<HSM_CLIENT_IMAGE_URL>
      auth:
        imagePullSecret: ""
    type: hsm
    version: v1
kind: ConfigMap
metadata:
  name: ibp-hsm-config
```

```
export NAMESPACE=<<NAMESPACE>>
kubectl apply -f ibp-hsm-config.yaml  -n ${NAMESPACE}
```

## Deployment Fabric component

The api-key (as the PIN) of Normal user must be provided during Fabric component deployment

The label of the slot must be provided during Fabric component deployment
