# SDK Direction for calling


## SDK in Python

### 1. Before calling

#### DApp Parameters
> DApp parameters are obtained from the Service Detail Page after the user participates in the app. Some parameters are set locally, which includes, 
 * __PCN gateway interface address:__ the calling address of the PCN (public city node) gateway
 * __User number：__ the number of the user
 * __Application number：__ number of participating applications
 * __Public key：__ the public key of PCN gateway downloaded after the user participates in the DApp
 * __Private key：__ A public key will be generated by BSN when DApps under Key-Trust Mode connects to the BSN successfully, and a private key will be generated corresponding to the public key uploaded for DApps under Public-Key-Upload Mode 
 * __Https certificate：__ the Https certificate used when the Https gateway interface is invoked

 #### Local Parameters
 * __Certificate Directory：__ the directory used to store the user's private key and certificate generated by DApps under Public-Key-Upload Mode when the user's certificate registration is invoked

### 2. Preparation

#### Import the SDK package
Introduce the following package 
```
from bsn_sdk_py.client.config import Config
from bsn_sdk_py.client.fabric_client import FabricClient
```
#### Initialize config
An object can be initialized to store all the configuration information, which should be introduced at the time of invocation after being configured or read by the caller. 
In config 'Init', the basic information of DApp is retrieved. Please do not call this operation frequently because this interface will occupy your TPS and traffic. 
You can use a static object to store 'config' in the project when you need it.   
When configuring the certificate, the certificate applied (that is, the certificate used for signing and verifying) is the direct certificate path, while the certificate for Https is the certificate root for the project. 
The file path to the directory is consistent with the previous example code.



```
	nodeApi = "" // PCN gateway address
	user_code:="" //User code
	app_code :="" //DApp code
	app_public_cert_path :="" //Public key path
	user_private_cert_path :="" //Private key path
	mspDir:="" //cert directory
	httpcert :="" //httpscert
	c = Config(user_code, app_code, nodeApi, mspDir, httpcert,
                 app_public_cert_path, user_private_cert_path)
```
#### Initialize Client
Use the generated configuration object and call the following code to create a Client object to invoke the PCN gateway
```
    client = FabricClient()
    client.set_config(c)
```

####   Call interface
Each gateway interface has encapsulated the parameter object of the request and response, which can be directly called just by assigning the value, and the operation of signing and verifying the signature has been realized in the function. 
The following is the call operation for the registered sub-user, and others are the same.
```
    client.register_user('hll4', '123456')
```

####   Log
To get a more detailed processing log, configure a logger.
```
        import logging
        FORMAT = "%(asctime)s %(thread)d %(message)s"
        logging.basicConfig(filename='bsn_test.log', filemode='w',level=logging.INFO, format=FORMAT, datefmt="[%Y-%m-%d %H:%M:%S]")

```

### 3.Other instructions

#### Description of the user identity certificate for DApp under Public-Key-Upload Mode   

Since the user certificate needed by DApp under Public-Key-Upload Mode when calling the PCN gateway for transaction needs to be generated by the user himself locally, the process is: registered user -> registered user certificate.
In the operation of registering a user certificate, a pair of keys are generated locally. A CSR file (the certificate or the application file) can be exported through the key. Call the user certificate, acquire a valid certificate via the registraion interface to process the transaction initiated by DApp under the Key-Trust Mode.
When setting CN in the CSR file, do not register Name directly, the Name is assembled by Name and AppCode in the format of 'nam@appcode'.  
This operation is made in the function of `GetCertificateRequest`   in   `bsn_sdk_py.client.entity.enroll_user.EnrollUser` . 

__Storage of certificates__ only in the form of local files at present,

Naming rules: 

          __stored cert__+ '\keystore\' + Name@AppCode + '_private.pem' 
          __stored cert__+ '\keystore\' + Name@AppCode + '_cert.pem'

#### About encryption 
In order to facilitate data encryption and decryption in the on-BSN operation of data transaction, a symmetric encryption 'AES' and an asymmetric encryption 'SM2' algorithm are implemented in the SDK 
Symmetric encryption for 'AES' is specifically called as follows:
```
	secret = '9999999999999999'
	t = "hello world"
    bsn = BsnAES(secret)
    e = bsn.encrypt(t)  # Encryption
    d = bsn.decrypt(e)  # Decryption
    assert t == d
```
Asymmetric encryption 'SM2', the details are as follows. In this function, both the signature and the verified signature of SM2 are made.

>Asymmetric encryption is encrypted by the public key and decrypted by the private key.
```
    from gmssl import sm2, func
	private_key = '00B9AB0B828FF68872F21A837FC303668428DEA11DCD1B24429D0C99E24EED83D5'
    public_key = 'B9C9A6E04E9C91F7BA880429273747D7EF5DDEB0BB2FF6317EB00BEF331A83081A6994B8993F3F5D6EADDDB81872266C87C018FB4162F5AF347B483E24620207'
    sm2_crypt = sm2.CryptSM2(
    public_key=public_key, private_key=private_key)

    #The data and encrypted data are generated in bytes. 
    data = b"111"
    enc_data = sm2_crypt.encrypt(data)
    dec_data =sm2_crypt.decrypt(enc_data)
    assert dec_data == data
```

#### About private key
In BSN, the encryption algorithm of 'fabric' framework is ECDSA secp256r1, while encryption algorithm of 'fisco-bcos' framework is' SM2'. 
When a uer participates in the DApp under Public-Key-Upload Mode, a key of the corresponding encryption algorithm needs to be generated and uploaded. 
  
Next is the description of how these two keys are generated. Keys are generated using 'openssl', where the generation of SM2 key requires version  1.1.1  of 'openssl' or above. 
> Note: the following commands are executed in a Linux environment.

##### 1. How the keys of ECDSA(secp256r1) are generated
- Generate a private key.
```
openssl ecparam -name prime256v1 -genkey -out key.pem
```
- Export the public key. 
```
openssl ec -in key.pem -pubout -out pub.pem
```
- Export the private key in pkcs8 format 
> Since it is convenient to use the key of pkcs8 format in some languages, you can export the pkcs8 format private key following the command below 
> The private key used in this SDK is in the format of pkcs8
```
openssl pkcs8 -topk8 -inform PEM -in key.pem -outform PEM -nocrypt -out key_pkcs8.pem
```
Three files can be generated from the command above.  
__`key.pem`__ :Private key  
__`pub.pem`__ :Public key  
__`key_pkcs8.pem`__ :Private key in pkcs8 format

##### 2.Generate a private key of `SM2` format   
First you need to check whether the version of 'openssl' supports' SM2 'format key generation using the following command.
```
openssl ecparam -list_curves | grep SM2
```
Support if the output shows as follows,
```
SM2       : SM2 curve over a 256 bit prime field
```
Otherwise you need to download version 1.1.1  or above from the official website.
This is used for version 1.1.1d.   
Download address: [https://www.openssl.org/source/openssl-1.1.1d.tar.gz](https://www.openssl.org/source/openssl-1.1.1d.tar.gz])  

- Generate a private key
```
openssl ecparam -genkey -name SM2 -out sm2PriKey.pem
```
- Export the public key
```
openssl ec -in sm2PriKey.pem -pubout -out sm2PubKey.pem
```
- Export the private key in pkcs8 format
> Since it is convenient to use the pkcs8 format key in some languages, you can export the pkcs8 format private key using the following command
> The private key used in this SDK is in the format of pkcs8
```
openssl pkcs8 -topk8 -inform PEM -in sm2PriKey.pem -outform pem -nocrypt -out sm2PriKeyPkcs8.pem
```
Three files can be generated from the above command  
__`sm2PriKey.pem`__ :Private key  
__`sm2PubKey.pem`__ :Public key  
__`sm2PriKeyPkcs8.pem`__ :Private key in pkcs8 format