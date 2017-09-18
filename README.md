https://technology.amis.nl/2017/07/04/ssltls-choose-cipher-suite/
For SSL/TLS connections, cipher suites determine for a major part how secure the connection will be. A cipher suite is a named combination of authentication, encryption, message authentication code (MAC) and key exchange algorithms used to negotiate the security settings (here). But what does this mean and how do you choose a secure cipher suite? The area of TLS is quite extensive and I cannot cover it in its entirety in a single blog post but I will provide some general recommendations based on several articles researched online. At the end of the post I’ll provide some suggestions for strong ciphers for JDK8.

Introduction

First I’ll introduce what a cipher suite is and how it is agreed upon by client / server. Next I’ll explain several of the considerations which can be relevant while making a choice of cipher suites to use.

What does the name of a cipher suite mean?

The names of the cipher suites can be a bit confusing. You see for example a cipher suite called: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384 in the SunJSSE list of supported cipher suites. You can break this name into several parts:

TLS: transport layer security (duh..)http://www.jscape.com/blog/cipher-suites
ECDHE: The key exchange algoritm is ECDHE (Elliptic curve Diffie–Hellman, ephemeral).
ECDSA: The authentication algorithm is ECDSA (Elliptic Curve Digital Signature Algorithm). The certificate authority uses an ECDH key to sign the public key. This is what for example Bitcoin uses.
WITH_AES_256_CBC: This is used to encrypt the message stream. (AES=Advanced Encryption Standard, CBC=Cipher Block Chaining). The number 256 indicates the block size.
SHA_384: This is the so-called message authentication code (MAC) algorithm. SHA = Secure Hash Algorithm. It is used to create a message digest or hash of a block of the message stream. This can be used to validate if message contents have been altered. The number indicates the size of the hash. Larger is more secure.
If the key exchange algorithm or the authentication algorithm is not explicitly specified, RSA is assumed. See for example here for a useful explanation of cipher suite naming.

What are your options

First it is a good idea to look at what your options are. This is dependent on the (client and server) technology used. If for example you are using Java 8, you can look here (SunJSSE) for supported cipher suites. In you want to enable the strongest ciphers available to JDK 8 you need to install Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files (here). You can find a large list of cipher suites and which version of JDK supports them (up to Java 8 in case of the Java 8 documentation). Node.js uses OpenSSL for cipher suite support. This library supports a large array of cipher suites. See here.

How determining a cipher suite works

They are listed in preference order. How does that work? During the handshake phase of establishing an TLS/SSL connection, the client sends supported cipher suites to the server. The server chooses the cipher to use based on the preference order and what the client supports.


This works quite efficiently, but a problem can arise when

There is no overlap in ciphers the client and server can speak
The only overlap between client and server supported cipher is a cipher which provides poor or no encryption
This is illustrated in the image below. The language represents the cipher suite. The order/preference specifies the encryption strength. In the first illustration, client and server can both speak English so the server chooses English. In the second image, the only overlapping language is French. French might not be ideal to speak but the server has no other choice in this case but to accept speaking French or to refuse talking to the client.



Thus it is a good practice to for the server only select specific ciphers which conform to your security requirements, but do of course take client compatibility into account.

How to choose a cipher suite

Basics

Check which cipher suites are supported

There are various mechanisms to check which ciphers are supported. For cloud services or websites you can use SSLLabs. For internal server checking, you can use various scripts available online such as this one or this one.

TLS 1.2

Of course you only want TLS 1.2 cipher suites since older TLS and SSL versions contain security liabilities. Within TLS 1.2 there is a lot to choose from. OWASP provides a good overview of which ciphers to choose here (‘Rule – Only Support Strong Cryptographic Ciphers’). Wikipedia provides a nice overview of (among other things) TLS 1.2 benefits such as GCM (Galois/Counter Mode) support which provides integrity checking.

Disable weak ciphers

As indicated before, if weak ciphers are enabled, they might be used, making you vulnerable. You should disable weak ciphers like those with DSS, DSA, DES/3DES, RC4, MD5, SHA1, null, anon in the name. See for example here and here. For example, do not use DSA/DSS: they get very weak if a bad entropy source is used during signing (here). For the other weak ciphers, similar liabilities can be looked up.

How to determine the key exchange algorithm

Types

There are several types of keys you can use. For example:

ECDHE: Use elliptic curve diffie-hellman (DH) key exchange (ephemeral). One key is used for every exchange. This key is generated for every request and does not provide authentication like ECDH which uses static keys.
RSA: Use RSA key exchange. Generating DH symetric keys is faster than RSA symmetric keys. DH also currently seems more popular. DH and RSA keys solve different challenges. See here.
ECDH: Use elliptic curve diffie-hellman key exchange. One key is for the entire SSL session. The static key can be used for authentication.
DHE: Use normal diffie-hellman key. One key is used for every exchange. Same as ECDHE but a different algorithm is used for the calculation of shared secrets.
There are other key algorithms but the above ones are most popular. A single server can host multiple certificates such as ECDSA and RSA certificates. Wikipedia is an example. This is not supported by all web servers. See here.

Forward secrecy

Forward secrecy means that is a private key is compromised, past messages which are send cannot also be decrypted. Read here. Thus it is beneficial to have perfect forward secrecy for your security (PFS).

The difference between ECDHE/DHE and ECDH is that for ECDH one key for the duration of the SSL session is used (which can be used for authentication) while with ECDHE/DHE a distinct key for every exchange is used. Since this key is not a certificate/public key, no authentication can be performed. An attacked can use their own key (here). Thus when using ECDHE/DHE, you should also implement client key validation on your server (2-way SSL) to provide authentication.

ECDHE and DHE give forward secrecy while ECDH does not. See here. ECDHE is significantly faster than DHE (here). There are rumors that the NSA can break DHE keys and ECDHE keys are preferred (here). On other sites it is indicated DHE is more secure (here). The calculation used for the keys is also different. DHE is prime field Diffie Hellman. ECDHE is Elliptic Curve Diffie Hellman. ECDHE can be configured. ECDHE-ciphers must not support weak curves, e.g. less than 256 bits (see here).

Certificate authority

The certificate authority you use to get a certificate from to sign the key can have limitations. For example, RSA certificates are very common while ECDSA is gaining popularity. If you use an internal certificate authority, you might want to check it is able to generate ECDSA certificates and use them for signing. For compatibility, RSA is to be preferred.

How to determine the message encryption mechanism

As a rule of thumb: AES_256 or above is quite common and considered secure. 3DES, EDE and RC4 should be avoided.

The difference between CBC and GCM

GCM provides both encryption and integrity checking (using a nonce for hashing) while CBC only provides encryption (here). You can not use the same nonce for the same key to encrypt twice when using GCM. This protects against replay attacks. GCM is supported from TLS 1.2.

How to choose your hashing algorithm

MD5 (here) and SHA-1 (here) are old and should not be used anymore. As a rule of thumb, SHA256 or above can be considered secure.

Finally

Considerations

Choosing a cipher suite can be a challenge. Several considerations play a role in making the correct choice here. Just to name a few;
Capabilities of server, client and certificate authority (required compatibility); you would choose a different cipher suite for an externally exposed website (which needs to be compatible with all major clients) than for internal security.

Encryption/decryption performance
Cryptographic strength; type and length of keys and hashes
Required encryption features; such as prevention of replay attacks, forward secrecy
Complexity of implementation; can developers and testers easily develop servers and clients supporting the cipher suite?
Sometimes even legislation plays a role since some of the stronger encryption algorithms are not allowed to be used in certain countries (we will not guess for the reason but you can imagine).

Recommendation

Based on the above I can recommend some strong cipher suites to be used for JDK8 in preference order:

TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDH_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDH_RSA_WITH_AES_256_GCM_SHA384
TLS_DHE_RSA_WITH_AES_256_GCM_SHA384
My personal preference would be to use TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 as it provides

Integrity checking: GCM
Perfect forward secrecy: ECDHE
Uses strong encryption: AES_256
Uses a strong hashing algorithm: SHA384
It uses a key signed with an RSA certificate authority which is supported by most internal certificate authorities.
Since ECDHE does not provide authentication, you should tell the server to verify client certificates (implement 2-way SSL).
