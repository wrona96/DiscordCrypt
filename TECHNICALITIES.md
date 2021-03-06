<h1 align="center">
    <img src="https://assets.gitlab-static.net/uploads/-/system/project/avatar/6431320/discordCrypt-mini.png" height="50" width="50">
    DiscordCrypt
    <h3>End-To-End Message Encryption For Discord</h3>
    <h5>Technical Details Overview</h5>
</h1>


* [**Preface**](#preface)
* [**Supported Algorithms**](#supported-algorithms)
    * [***Ciphers***](#ciphers)
        * [*Cipher Modes*](#cipher-modes)
        * [*Message Padding*](#message-padding)
    * [***Key Exchanges***](#key-exchanges)
        * [*Diffie-Hellman*](#diffie-hellman)
        * [*Elliptic Curve Diffie-Hellman*](#elliptic-curve-diffie-hellman)
    * [***Hash Algorithms***](#hash-algorithms)
        * [*Scrypt Hashing Algorithm*](#scrypt-hashing-algorithm)
        * [*SHA3 Hashing Algorithm*](#sha3-hashing-algorithm)
* [**Message Format**](#message-format)
    * [***Meta-Data Encoding***](#meta-data-encoding)
    * [***User Message Format***](#user-message-format)
        * [*Message Authentication*](#user-message-authentication)
    * [***Public Key Format***](#public-key-format)
    * [***Byte Encoding***](#byte-encoding)
* [**General Encryption & Decryption Process**](#general-encryption-and-decryption-process)
* [**Master Database Encryption**](#master-database-encryption)
* [**Key Exchange Process**](#key-exchange-process)
* [**Known Vulnerabilities**](#known-vulnerabilities)

# Preface

This guide assumes the reader has basic knowledge of general cryptography including a top-level 
    understanding of ***ciphers***, ***message padding***, ***hashing***, ***key exchanging*** and 
    ***authentication*** processes.

While this guide attempts to provide an overview of the processes involved in the plugin, it 
    should not be relied on to reflect the **current implementation** of the cryptographic processes.

Discord currently runs on an extremely outdated version of NodeJS ( `v7.4.0` ) which further limits 
    implementation capabilities. While we have tried to get past this, the plugin is greatly limited 
    by this factor and should be taken into account while evaluating these methods.

A final note, this plugin was developed to be as secure as possible at the expense of speed and 
    increased complexity. While multiple implementations of a given cryptographic suite may be 
    supported, by default, the plugin chooses the maximum security-based options for operation.

# Supported Algorithms

**DiscordCrypt** uses a variety of symmetric encryption algorithms exposed via 
    [NodeJS](https://en.wikipedia.org/wiki/Node.js)'s 
    [`crypto`](https://nodejs.org/api/crypto.html) module.

In addition to these, two types of key exchanges are supported as well as the ability to generate 
    [random numbers](https://nodejs.org/api/crypto.html#crypto_crypto_randombytes_size_callback) 
    which is also exposed by NodeJS.

Every algorithm used in this plugin is directly provided by NodeJS. The only things manually 
    implemented are:

* [Cipher Padding Modes](#message-padding)
* [Scrypt Hashing Algorithm](https://en.wikipedia.org/wiki/Scrypt) via Asm.js

**DiscordCrypt** also relies on several libraries which may or may not have been tweaked to work 
    within the Discord client. These are indicated below.

* [OpenPGP.js](https://openpgpjs.org) ( License: [LGPL](https://www.gnu.org/copyleft/lesser.html) )
* [SJCL](https://crypto.stanford.edu/sjcl) ( License: [BSD](https://opensource.org/licenses/BSD-2-Clause) )
* [SIDH.js](https://github.com/cyph/sidh.js) ( License: [MIT](https://opensource.org/licenses/MIT) )
* [Currify](https://www.npmjs.com/package/currify) ( License: [MIT](https://opensource.org/licenses/MIT) )
* [JS-SHA3](https://www.npmjs.com/package/js-sha3) ( License: [MIT](https://opensource.org/licenses/MIT) )
* [Smalltalk](https://www.npmjs.com/package/smalltalk) ( License: [MIT](https://opensource.org/licenses/MIT) )
* [Curve25519-JS](https://github.com/wavesplatform/curve25519-js) ( License: [MIT](https://opensource.org/licenses/MIT) )

#### Ciphers

The following ciphers are currently supported by **DiscordCrypt**.

 * [Camellia-256](https://en.wikipedia.org/wiki/Camellia_(cipher)) ( **Default Primary Cipher** ) 
    [  [Source](https://leogx9r.gitlab.io/DiscordCrypt/-_discordCrypt.html#.camellia256_encrypt)  ]
 * [AES-256 ( Rijndael )](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) ( **Default Secondary Cipher** )
    [  [Source](https://leogx9r.gitlab.io/DiscordCrypt/-_discordCrypt.html#.aes256_encrypt)  ]
 * [TripleDES-192](https://en.wikipedia.org/wiki/Triple_DES)
    [  [Source](https://leogx9r.gitlab.io/DiscordCrypt/-_discordCrypt.html#.tripledes192_encrypt)  ]
 * [IDEA-128](https://en.wikipedia.org/wiki/International_Data_Encryption_Algorithm)
    [ [Source](https://leogx9r.gitlab.io/DiscordCrypt/-_discordCrypt.html#.idea128_encrypt) ]
 * [Blowfish-512](https://en.wikipedia.org/wiki/Blowfish_(cipher))
    [ [Source](https://leogx9r.gitlab.io/DiscordCrypt/-_discordCrypt.html#.blowfish512_encrypt) ]

##### Cipher Modes

Each cipher can operate in a number of 
    [block operation modes](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation).

The currently supported modes are:

- [Cipher Block Chaining](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_Block_Chaining_(CBC)) 
    ( **Default Block Mode** )
- [Cipher Feedback Mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_Feedback_(CFB))
- [Output Feedback Mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Output_Feedback_(OFB))

##### Message Padding

For each cipher, messages must be 
    [cryptographically padded](https://en.wikipedia.org/wiki/Padding_(cryptography)) 
    to meet the cipher's block size.

***While NodeJS supports some padding schemes, most are not. As such, 
    these were manually implemented in the code.***

The following padding schemes are supported:

* [PKCS #5/PKCS #7](https://en.wikipedia.org/wiki/Padding_(cryptography)#PKCS7) 
    ( **Default Padding Scheme** )
* [ANSI x9.23](https://en.wikipedia.org/wiki/Padding_(cryptography)#ANSI_X.923)
* [ISO 10126](https://en.wikipedia.org/wiki/Padding_(cryptography)#ISO_10126)
* [ISO 97971](https://en.wikipedia.org/wiki/Padding_(cryptography)#ISO/IEC_7816-4)

These methods have all been manually implemented 
    [here](https://leogx9r.gitlab.io/DiscordCrypt/-_discordCrypt.html#.__padMessage).

#### Key Exchanges

> Key exchange (also key establishment) is any method in 
    [cryptography](https://en.wikipedia.org/wiki/Cryptography) by which 
    [cryptographic keys](https://en.wikipedia.org/wiki/Cryptographic_keys) 
    are exchanged between two parties, allowing use of a cryptographic 
    [algorithm](https://en.wikipedia.org/wiki/Algorithm).
>
> If the sender and receiver wish to exchange encrypted messages, each must be equipped 
    to encrypt messages to be sent and decrypt messages received. The nature of the equipping 
    they require depends on the [encryption](https://en.wikipedia.org/wiki/Algorithm) technique 
    they might use. If they use a [code](https://en.wikipedia.org/wiki/Code_(cryptography)), 
    both will require a copy of the same [codebook](https://en.wikipedia.org/wiki/Codebook). 
    If they use a [cipher](https://en.wikipedia.org/wiki/Cipher), they will need appropriate 
    keys. 
>
> If the cipher is a [symmetric key cipher](https://en.wikipedia.org/wiki/Symmetric_key), 
    both will need a copy of the same key. If an 
    [asymmetric key cipher](https://en.wikipedia.org/wiki/Public_key) with the public/private 
    key property, both will need the other's public key. 

The following algorithms are currently supported to exchange keys in a secure manner.

* [Diffie-Hellman ( ***DH*** )](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)
* [Elliptic Curve Diffie-Hellman ( ***ECDH*** ) ](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman) 
    ( **Default Exchange Algorithm** )

The default type and key size was chosen to provide roughly 256-bits of security.
 This is further detailed below.


    The table below was taken from SP800-57, Recommendation for Key Management, Section 5.6.1.
    
    In the table below, 2TDEA is 2-key triple-DES; and 3TDEA is 3-key triple-DES and sometimes 
    referred to as just triple DES. Triple DES is specified in SP800-67,
    
    Recommendation for the Triple Data Encryption Algorithm (TDEA) Block Cipher.
    The yellow and green highlights are explained in the NIST Recommendations section. 
    
    Source: https://www.cryptopp.com/wiki/Security_Level#Finite_Field 
    Publication: https://csrc.nist.gov/csrc/media/publications/sp/800-131a/rev-1/final/documents/sp800-131a_r1_draft.pdf


- ----------------- - ----------------- - ------------- - ---------- - ------------- -
| **Security Bits** | **Symmetric Key** |    **FF**     |   **IF**   |    **EC**     |
| ----------------- | ----------------- | ------------- | ---------- | ------------- |
|        80         |        2TDEA      | L=1024  N=160 |   k=1024   |   f=160-223   |
|       112         |        3TDEA      | L=2048  N=224 |   k=2048   |   f=224-255   |
|       128         |         AES       | L=3072  N=256 |   k=3072   |   f=256-383   |
|       192         |         AES       | L=7680  N=384 |   k=7680   |   f=384-511   |
|       256         |         AES       | L=15360 N=511 |   k=15360  |   f=512+      |
- ----------------- - ----------------- - ------------- - ---------- - ------------- -


    **Security Bits**

        Security Bits estimate the computational steps or operations (not machine instructions) 
         required to find a solution to the problem in the problem's domain (FF, IF, or EC).

        For example, if someone says, 'My system uses 1024 Diffie Hellman", they are really
         stating their system has a security level of 80 bits (and because its Diffie Hellman,
         the problem domain is finite field). It will take a computer, on average, approximately 
         280 operations to find a solution (think Big-Oh notation).

        To break Diffie-Hellman via classical discrete logarithms, a number of methods could be
         employed: Index calculus, modified Pollard's rho, or Baby-step giant-step to name a few.

    Symmetric Key

        Symmetric Key is a block cipher algorithm that offers an equivalent strength.

        Though DES and AES are listed, any non-wounded or non-broken block cipher can be used.

        For example, European and international users might want to use Cameilla rather than AES 
         since Cameilla is NESSIE and ISO approved. Note that an appropriate mode
         ( block cipher mode of operation ) must also be chosen.

    Finite Field ( FF )

        FF is finite field cryptography, sometimes referred to as the discrete log problem (DLP), 
        and examples include Diffie-Hellman, ElGamal, and DSA ( one of the three signature schemes 
        specified in the Digital Signature Standard (DSS) ).

        L is the size of the field, N is the size of the subgroup.

    Integer Factorization ( IF )

        IF is integer factorization and is the underlying problem of RSA and Rabin-Williams.

    Elliptic Curves ( EC )

        EC is elliptical curve cryptography. 



##### Diffie-Hellman

> **Diffie–Hellman key exchange** (**DH**)
    [[1]](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange#cite_note-1) is a 
    method of securely exchanging 
    [cryptographic keys](https://en.wikipedia.org/wiki/Key_(cryptography)) over a public 
    channel and was one of the first
    [public-key protocols](https://en.wikipedia.org/wiki/Public-key_cryptography) as 
    originally conceptualized by
    [Ralph Merkle](https://en.wikipedia.org/wiki/Ralph_Merkle "Ralph Merkle") and named after
    [Whitfield Diffie](https://en.wikipedia.org/wiki/Whitfield_Diffie "Whitfield Diffie") and
    [Martin Hellman](https://en.wikipedia.org/wiki/Martin_Hellman "Martin Hellman").

This exchange algorithm supports a large number of key sizes and operates on a similar 
    principle to [RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem)), 
    an asymmetric encryption algorithm.

The main advantage of this algorithm is it is generally considered secure 
    ( as long as the key size is sufficiently large ) but its downside is as the key size 
    increases, the time taken to generate a key pair increases proportionately.

One should generally choose the largest key size when exchanging keys for maximum privacy 
    at the expense of speed.

The following key sizes in bits are supported:

* 768  Bits ( `modp1`  )
* 1024 Bits ( `modp2`  )
* 1536 Bits ( `modp5`  )
* 2048 Bits ( `modp14` )
* 3072 Bits ( `modp15` )
* 4096 Bits ( `modp16` )
* 6144 Bits ( `modp17` )
* 8192 Bits ( `modp18` )

Please see [here](https://www.rfc-editor.org/rfc/rfc2412.txt) for parameters defined less 
    than 2048 or [here](https://www.rfc-editor.org/rfc/rfc3526.txt) for parameters larger than 
    2048 bits.

##### Elliptic Curve Diffie-Hellman

> **Elliptic-curve Diffie–Hellman** (**ECDH**) is an anonymous 
    [key agreement](https://en.wikipedia.org/wiki/Key_agreement) 
    protocol that allows two parties, each having an elliptic-curve public–private 
    key pair, to establish a 
    [shared secret](https://en.wikipedia.org/wiki/Shared_secret) over an 
    [insecure channel](https://en.wikipedia.org/wiki/Insecure_channel).
    [[1]](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman#cite_note-1) 
    [[2]](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman#cite_note-2) 
    [[3]](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman#cite_note-3)
>   This shared secret may be directly used as a key, or to 
    [derive another key](https://en.wikipedia.org/wiki/Key_derivation_function).
>   The key, or the derived key, can then be used to encrypt subsequent communications using a 
    [symmetric-key cipher](https://en.wikipedia.org/wiki/Symmetric-key_algorithm).
>   It is a variant of the 
    [Diffie–Hellman](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)  
    protocol using 
    [elliptic-curve cryptography](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography).

This operates on the same principle of a Diffie-Hellman exchange except it is done using 
    Elliptic Curves.

The main benefits of this algorithm is that it is much smaller in size in contrast to 
     standard Diffie-Hellman keys and is extremely quick to generate a key pair while 
     offering the same level of security..

The downside, however, is that many consider it insecure as concerns of various 
    [backdoors](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography#Backdoors) 
    have been brought forth. While these concerns relate to a random-bit generator 
    ( [DUAL_EC_DRBG](https://en.wikipedia.org/wiki/Dual_EC_DRBG) ) which is not directly 
    related to the general use of elliptic curves, there has been enough mistrust placed 
    by prominent agencies to generally refrain from usage of these.

Just as with Diffie-Hellman, one should choose the largest key size for key exchanges as 
    the security of the key size is equivalent to a Diffie-Hellman keys with a much faster 
    generation speed. 

The following key sizes in bits are supported:

* 224 Bits ( `secp224k1: SECG curve over a 224 bit prime field.` )
* *256 Bits* ( `x25519: Unpatented curve over a 256 bit prime field.` )
* 384 Bits ( `secp384r1: NIST/SECG curve over a 384 bit prime field.` )
* 409 Bits ( `sect409k1: NIST/SECG curve over a 409 bit binary field.` )
* 521 Bits ( `secp521r1: NIST/SECG curve over a 521 bit prime field.` )
* 571 Bits ( `sect571k1: NIST/SECG curve over a 571 bit binary field.` )
* *751 Bits* ( `sidhp751: Post-Quantum Supersingular Isogeny Diffie-Hellman over a 751 bit prime field.` ) ( **Default** )

Please see [here](http://www.secg.org/sec2-v2.pdf) for more information on curve standard parameters.

For information regarding the specifics of X25519, see its introductory paper 
    [here](https://cr.yp.to/ecdh/curve25519-20060209.pdf).

With the advancements of quantum computers, the need for 
    [Post-Quantum Cryptography](https://en.wikipedia.org/wiki/Post-quantum_cryptography) is actively 
    being explored. Several such methods have already been proposed to be "quantum resistant".
    
The curve above defined as [`sidhp751`](https://eprint.iacr.org/2016/413.pdf) is a specially chosen  
    [supersingular elliptic curve](https://en.wikipedia.org/wiki/Supersingular_elliptic_curve) 
    with [isogenic](https://en.wikipedia.org/wiki/Supersingular_isogeny_graph) properties used in the familiar 
    Diffie-Hellman protocol. In post quantum cryptography, 
    "[Supersingular Isogeny Diffie-Hellman](https://en.wikipedia.org/wiki/Supersingular_isogeny_key_exchange)" 
    is said to be quantum resistant while still benefiting from the standard ECC property of retaining a  
    small fingerprint. This curve offers `128 quantum bits` of security.

#### Hash Algorithms

**DiscordCrypt** internally uses the `scrypt` hashing method for deriving keys from passwords and SHA3 
    for other requirements.

##### Scrypt Hashing Algorithm

The [Scrypt](https://en.wikipedia.org/wiki/Scrypt) algorithm is a key-derivative function that was 
designed to be "memory hard" in such a way that it is dependant on access to fast memory. 
    This design choice makes it desirable for use in password-based calculations.

Due to Node's crypto module not natively supporting this, it was manually implemented in Asm.js for 
    deriving an AES-256 bit key for encrypting the master database.

The original source for this was taken from [here](https://github.com/Tarsnap/scrypt) and was 
    transpiled to Asm.js for faster execution.

Scrypt uses a total of 6 parameters. Each of these will be described below.

    # Taken from the blog: https://blog.filippo.io/the-scrypt-parameters/

    > Parameter: 𝑟 - Memory Tuner
        
    BlockMix turns a hash function with 𝑘-bit long inputs and outputs into a hash function 
    with 2𝑟𝑘-bit long inputs and outputs. That is, it makes the core hash function in 
    scrypt 2𝑟 wider.
    
    It does that by iterating the hash function 2𝑟 times, so both memory 
    usage (to store the hash values) and CPU time scale linearly with it. That is, 
    if 𝑟 doubles the resources double.
    
    That's useful because scrypt applies the hash to "random" memory positions. 
    CPUs load memory in fixed-size blocks called cache lines. If the hash block size is 
    smaller than the cache line, all the rest of the loaded line will be wasted memory 
    bandwidth. Also, it dilutes the memory latency cost. Percival predicted both cache line 
    sizes and memory latencies would increase over time, so made the hash size tunable to 
    prevent scrypt from becoming latency-bound.
    
    
    > Parameter: N - Work Factor
    
    Memory and CPU usage scale linearly with 𝑁. The mixing function, ROMix, stores 𝑁 
    sequential hash results in RAM, to then load them in a random order and sequentially 
    xor and hash them.
    
    The reason 𝑁 must be a power of two is that to randomly select one of the 𝑁 memory 
    slots at each iteration, scrypt converts the hash output to an integer and reduces 
    it mod 𝑁. If 𝑁 is a power of two, that operation can be optimized into simple (and fast) 
    binary masking.
    
    > Parameter: 𝑝 - Parallel Factor
    
    𝑝 is used in the outmost function, MFcrypt. It is a parallelization parameter. 
    𝑝 instances of the mixing function are run independently and their outputs concatenated 
    as salt for the final PBKDF2.
    
    𝑝 > 1 can be handled in two ways: sequentially, which does not increase memory usage but 
    requires 𝑝 times the CPU and wall clock time; or parallelly, which requires 𝑝 times the 
    memory and effective CPU time, but does not increase wall clock time.
    
    So 𝑝 can be used to increase CPU time without affecting memory requirements when handled 
    sequentially, or without affecting wall clock time when handled parallelly. However, it 
    offers attackers the same opportunity to optimize for processing or memory.
    
    > Parameter: dkLen - Length of output
    
    This parameter indicates the desired length of the output key derived from the input and 
    salt. It should be at least 256 bits due to that being the minimum output of the SHA-256 
    algorithm which is internally used for final compression.
    
    This must satisfy: dkLen ≤ (2^32− 1)
    
    > Parameter: input - The input value
    
    This parameter is self-explanatory. This is the input value which is used to derive the 
    initial block data and also used in the final PBKDF2 compression operation.
    
    > Parameter: salt - The secret value
    
    This parameter is used as the initial secret for calculation of the initial state of the 
    block along with the input value.
    
    Estimating Scrypt Memory Usage
    
    scrypt requires 𝑁 times the hash block size memory. Because of BlockMix, the hash block 
    size is 2𝑟 the underlying hash output size. In scrypt, that hash is the Salsa20 core, 
    which operates on 64-bytes blocks.
    
    So the minimum memory requirement of scrypt is:
    
    𝑁 × 2𝑟 × 64 = 128 × 𝑁 × 𝑟 bytes
    
    For 𝑁 = 16384 and 𝑟 = 8 that would be 16 MiB. It scales linearly with 𝑁 and 𝑟, and some 
    implementations or APIs might cause internal copying doubling the requirement.

You may find the actual implementation of it 
    [here](https://leogx9r.gitlab.io/DiscordCrypt/-_discordCrypt.html#.scrypt).

##### SHA3 Hashing Algorithm

The general implementation for SHA3 was implemented via the JS-SHA3 library which can be found 
    [here](https://github.com/emn178/js-sha3).

In particular, the following methods have been implemented whose demos are below:

- [SHA3-512](http://emn178.github.io/online-tools/sha3_512.html)  
- [SHA3-384](http://emn178.github.io/online-tools/sha3_384.html)  
- [SHA3-256](http://emn178.github.io/online-tools/sha3_256.html)  
- [SHA3-224](http://emn178.github.io/online-tools/sha3_224.html)  
- [Keccak-512](http://emn178.github.io/online-tools/keccak_512.html)  
- [Keccak-384](http://emn178.github.io/online-tools/keccak_384.html)  
- [Keccak-256](http://emn178.github.io/online-tools/keccak_256.html)  
- [Keccak-224](http://emn178.github.io/online-tools/keccak_224.html)  
- [Shake128](http://emn178.github.io/online-tools/shake_128.html)  
- [Shake256](http://emn178.github.io/online-tools/shake_256.html)  

Additionally, the Keccak ( SHA3 ) family also has a 
    [MAC](https://en.wikipedia.org/wiki/Message_authentication_code) function implemented in 
    this library under the name of KMAC ( Keccak-MAC ) providing authentication tags of either 
    128 bits or 256-bits.

Internally, the KMAC method is used mainly to derive encryption keys for ciphers by combining the 
    password with a 64 bit unique salt.

### Meta Data Encoding

Each user message contains 4 characters of [encoded](#byte-encoding) metadata.

These decode to a 32-bit integer encoded in Little-Endian order whose byte positions 
    indicates the data type.


| **Byte Position** | **Description**                                                                   |
| ----------------- | --------------------------------------------------------------------------------- |
| `0`               | Details symmetric ciphers used to encrypt the message.                            |
| `1`               | Indicates the block operation mode of the symmetric cipher.                       |
| `2`               | Contains the padding scheme used to align the message to the cipher's block size. |
| `3`               | Contains a random byte. ( `Reserved For Future Use` )                             |

##### Cipher Indexing

Cipher indexes are based on the combination of ciphers used for encrypting a message.

Since each message undergoes two encryption or decryption processes, an index is assigned 
    to indicate which combination of ciphers are used. This index is attached to the metadata in 
    each encrypted message payload in the form of an 8-bit word.

Below indicates the current index assignment for each combination of ciphers.  

```javascript
/**
 * @desc Indexes of each dual-symmetric encryption mode.
 * @type {int[]}
 */
_encryptModes = [
    /* Blowfish(Blowfish, AES, Camellia, IDEA, TripleDES) */
    0, 1, 2, 3, 4,
    /* AES(Blowfish, AES, Camellia, IDEA, TripleDES) */
    5, 6, 7, 8, 9,
    /* Camellia(Blowfish, AES, Camellia, IDEA, TripleDES) */
    10, 11, 12, 13, 14,
    /* IDEA(Blowfish, AES, Camellia, IDEA, TripleDES) */
    15, 16, 17, 18, 19,
    /* TripleDES(Blowfish, AES, Camellia, IDEA, TripleDES) */
    20, 21, 22, 23, 24
];
```

## Message Format

A message can consist in two forms.

- User Encrypted Message
- Public Key Message

Each message can be determined by the first 4 characters which is a unique magic string 
    indicating its type.


| **Type**             | **Magic**   |
| -------------------- | ----------- |
| `User Message`       | `⢷⢸⢹⢺`      |
| `Public Key Message` | `⢻⢼⢽⢾`      |

### User Message Format

A user message is expressed in the following format:

**N.B** Character size refers to the UTF-16 character position of a 
    [byte encoded](#byte-encoding) message.


| **Character Size** | **Field**    | **Description**                                               |
| ------------------ | ------------ | ------------------------------------------------------------- |
| `4`                | Magic        | Indicates an encrypted message.                               |
| `4`                | Metadata     | Details the message format. See [here](#meta-data-encoding).  |
| `< Variable >`     | Message      | Contains the encrypted ciphertext.                            |

#### User Message Authentication

All user messages contain a [KMAC](https://en.wikipedia.org/wiki/HMAC) tag prepended to it.

This KMAC uses SHA3 along with the primary message key to form a hash of the outer 
    ciphertext of the message.

This is prepended such that the variable length message now follows the following format:


| **Byte Position**  | **Byte Size**    | **Description**                                               |
| ------------------ | ---------------- | ------------------------------------------------------------- |
| `0`                | `32`             | Contains the 256-bit HMAC tag of the ciphertext.              |
| `32`               | `Variable`       | Contains the ciphertext ( Including a unique 64-bit salt ).   |


This tag is used for authentication to ensure ciphertexts have not been tampered with 
    during transit.

In addition to this, during verification, they are compared in a time-safe manner to 
    prevent possible forms of [timing attacks](https://en.wikipedia.org/wiki/Timing_attack) 
    even though they're not required in this use case.

### Public Key Format

In contrast to a [user message](#user-message-format), a public key message is expressed as follows:


| **Character Size** | **Field**        | **Description**                                  |
| ------------------ | ---------------- | ------------------------------------------------ |
| `4`                | Magic            | Indicates an encrypted message.                  |
| `1`                | Algorithm Type   | Indicates the algorithm and bit size of the key. |
| `1`                | Salt Length      | Indicates the length of the user salt.           |
| `< Variable >`     | User Salt        | A randomly generated series of bytes.            |
| `< Variable >`     | Public Key       | Contains the raw public key.                     |

### Byte Encoding

While all messages are Base64 or hex encoded, Discord does not use a monospace font. This allows 
    messages to look uneven when sent.

To combat this, a simple method of substitution is used to replace all characters 
    in their hex-based representation with a 256-character monospace-type width using the 
    [Braille character set](https://en.wikipedia.org/wiki/Braille_Patterns).

Discord itself treats both UTF-8 and UTF-16 characters as the same length ( character-limit-wise ) 
    meaning both a UTF-8 and UTF-16 messages are limited to 2000 characters each.

This is handled by the methods `__substituteMessage`, `__metaDataEncode` and `__metaDataDecode`.

These methods do a 1-1 substitution as follows:


| **Character Set** | **String**                                                                                      |
| ----------------- | --------------------------------------------- |
| `Base 64`           | `000102030405060708` ... `F8F9FAFBFCFDFEFF` |
| `Braille`           | `⠀⠁⠂⠃⠄⠅⠆⠇` ... `⣸⣹⣺⣻⣼⣽⣾⣿`                |

The raw code for this is defined in the `__getBraille()` function seen below.

```javascript
return Array.from(
    "⠀⠁⠂⠃⠄⠅⠆⠇⠈⠉⠊⠋⠌⠍⠎⠏⠐⠑⠒⠓⠔⠕⠖⠗⠘⠙⠚⠛⠜⠝⠞⠟⠠⠡⠢⠣⠤⠥⠦⠧⠨⠩⠪⠫⠬⠭⠮⠯⠰⠱⠲⠳⠴⠵⠶⠷⠸⠹⠺⠻⠼⠽⠾⠿⡀⡁⡂⡃⡄⡅⡆⡇⡈⡉⡊⡋⡌⡍⡎⡏⡐⡑⡒⡓⡔⡕⡖" +
    "⡗⡘⡙⡚⡛⡜⡝⡞⡟⡠⡡⡢⡣⡤⡥⡦⡧⡨⡩⡪⡫⡬⡭⡮⡯⡰⡱⡲⡳⡴⡵⡶⡷⡸⡹⡺⡻⡼⡽⡾⡿⢀⢁⢂⢃⢄⢅⢆⢇⢈⢉⢊⢋⢌⢍⢎⢏⢐⢑⢒⢓⢔⢕⢖⢗⢘⢙⢚⢛⢜⢝⢞⢟⢠⢡⢢⢣⢤⢥⢦⢧⢨⢩⢪⢫⢬⢭" +
    "⢮⢯⢰⢱⢲⢳⢴⢵⢶⢷⢸⢹⢺⢻⢼⢽⢾⢿⣀⣁⣂⣃⣄⣅⣆⣇⣈⣉⣊⣋⣌⣍⣎⣏⣐⣑⣒⣓⣔⣕⣖⣗⣘⣙⣚⣛⣜⣝⣞⣟⣠⣡⣢⣣⣤⣥⣦⣧⣨⣩⣪⣫⣬⣭⣮⣯⣰⣱⣲⣳⣴⣵⣶⣷⣸⣹⣺⣻⣼⣽⣾⣿"
);
```

## General Encryption And Decryption Process

Encryption and decryption follows [OpenSSL](https://en.wikipedia.org/wiki/OpenSSL)'s 
    method of deriving keys.

A random 64-bit salt is generated and is used in conjunction with a `KMAC` to generate a unique 
    [Initialization Vector](https://en.wikipedia.org/wiki/Initialization_vector) and 
    a derived encryption key.

This is used to prevent the same message being encrypted multiple times over the course of 
    a conversation from having the same ciphertext being produced.

Messages encrypted in this format take the form:


| **Byte Position**  | **Byte Size**    | **Description**                                               |
| ------------------ | ---------------- | ------------------------------------------------------------- |
| `0`                | `8`              | Contains a one time salt used to generate a unique key.       |
| `8`                | `Variable`       | Contains the ciphertext.                                      |

When a message is being decrypted, the [metadata](#meta-data-encoding) for the message is 
    read to determine how  to proceed.

This indicates:

- The symmetric ciphers used.
- The block operation mode of the ciphers.
- The padding scheme used for the message.

All inputs passed to a `< cipher >_decrypt` method is assumed to contain a 64-bit seed 
    used to derive the key and IV.

The plugin employs what is known as 
    [multiple encryption](https://en.wikipedia.org/wiki/Multiple_encryption) to encrypt 
    all messages before they are sent.

For this, two ciphers are used. The primary cipher is used to encrypt the 
    [plaintext](https://en.wikipedia.org/wiki/Plaintext) message into 
    [ciphertext](https://en.wikipedia.org/wiki/Ciphertext).

Following this, the secondary cipher is used along with a completely different key to 
    encrypt the ciphertext yet again yielding the final ciphertext.

Finally, a HMAC authentication tag using 
    [KMAC](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-185.pdf) 
    is computed on the final ciphertext with the concatenated primary and secondary keys and 
    is prepended to it, thus creating the final result.

This result is then encoded to [base 64](https://en.wikipedia.org/wiki/Base64) and undergoes 
    [byte encoding](#byte-encoding) to produce a monospace-compatible message.


## Putting It Together

A general user message is as follows:


```
| ------------------------- ENCRYPTED MESSAGE PAYLOAD ------------------------- |
|                                                                               |
|  | --------- | | -------- | | -------- | | ------------------------------- |  |
|  |           | |          | |          | |                                 |  |
|  |           | |          | |          | |                                 |  |
|  |   MAGIC   | | METADATA | |   KMAC   | |         CIPHERTEXT BLOB         |  |
|  |           | |          | |          | |                                 |  |
|  |  4 Bytes  | | 8 Bytes  | | 32 Bytes | |          < VARIABLE >           |  |
|  |           | |          | |          | |                                 |  |
|  |           | |          | |          | |                                 |  |
|  | --------- | | -------- | | -------- | | ------------------------------- |  |
|                                                                               |
| ----------------------------------------------------------------------------- |
```

- `MAGIC`
    - Contains the 4 magic characters indicating an encrypted message payload.

- `METADATA`
    - Encoded metadata indicates the ciphers used, padding scheme and block operation.

- `KMAC`
    - Contains a KECCAK-256 MAC of the `CIPHERTEXT BLOB` object.

- `CIPHERTEXT BLOB`
    - Contains an encrypted message blob detailed below.
    
    
```
| ------------------------------ CIPHERTEXT BLOB ----------------------------- |
|                                                                              |
|   | ---------------------- OUTER CIPHERTEXT DATA ----------------------- |   |
|   |                                                                      |   |
|   |  | ---------- | | ----------------------------------------------- |  |   |
|   |  |            | |                                                 |  |   |
|   |  |            | |                                                 |  |   |
|   |  |            | |                                                 |  |   |
|   |  |   NONCE    | |              INNER CIPHERTEXT BLOB              |  |   |
|   |  |            | |                                                 |  |   |
|   |  |  8 Bytes   | |                  < VARIABLE >                   |  |   |
|   |  |            | |                                                 |  |   |
|   |  |            | |                                                 |  |   |
|   |  |            | |                                                 |  |   |
|   |  | ---------- | | ----------------------------------------------- |  |   |
|   |                                                                      |   |
|   | -------------------------------------------------------------------- |   |
|                                                                              |
| ---------------------------------------------------------------------------- |
```

- `OUTER CIPHERTEXT DATA`
    - Contains a nonce concatenated with an encrypted blob.

- `NONCE`
    - Contains a random 64-bit nonce used for KDF decryption of the INNER CIPHERTEXT DATA.

- `INNER CIPHERTEXT BLOB`
    - Contains an encrypted `CIPHERTEXT DATA` object from using the **secondary key and nonce**.


```
| --------------------------- INNER CIPHERTEXT BLOB -------------------------- |
|                                                                              |
|   | ---------------------- INNER CIPHERTEXT DATA ----------------------- |   |
|   |                                                                      |   |
|   |  | ---------- | | ----------------------------------------------- |  |   |
|   |  |            | |                                                 |  |   |
|   |  |            | |                                                 |  |   |
|   |  |            | |                                                 |  |   |
|   |  |   NONCE    | |                    MESSAGE                      |  |   |
|   |  |            | |                                                 |  |   |
|   |  |  8 Bytes   | |                  < VARIABLE >                   |  |   |
|   |  |            | |                                                 |  |   |
|   |  |            | |                                                 |  |   |
|   |  |            | |                                                 |  |   |
|   |  | ---------- | | ----------------------------------------------- |  |   |
|   |                                                                      |   |
|   | -------------------------------------------------------------------- |   |
|                                                                              |
| ---------------------------------------------------------------------------- |
```

- `INNER CIPHERTEXT DATA`
    - Contains a nonce concatenated with an encrypted blob.

- `NONCE`
    - Contains a random 64-bit nonce used for KDF decryption of the MESSAGE.

- `MESSAGE`
    - Contains the encrypted message using the **primary key and nonce**.


This message is sent to Discord's servers in an embedded message with additional 
    aesthetic icons to produce the image seen below. It may optionally be sent in 
    the form of a code block in the case that embeds are not used.
 

![An encrypted message](images/encrypted-message.png)

## Master Database Encryption

The master database uses an AES-256 bit key for encryption and decryption in `GCM` mode 
    derived from the password the user inputs. The database is first converted to a 
    string via `JSON.stringify` and then padded using the `PKCS #7` scheme.

Its derivation is done by using the [Scrypt](#scrypt-hashing-algorithm) 
    hashing algorithm with the following parameters.


| **Parameter** | **Value** |
| ------------- | --------- |
| `N`           | 16384     |
| `r`           | 16        |
| `p`           | 1         |
| `dkLen`       | 32        |

**N.B. Scrypt methods cannot be run in parallel in this implementation so they are run 
    single-threaded.**

This derives a 256-bit key which is used in conjunction with the 
    `__aes256_encrypt_gcm`/`__aes256_decrypt_gcm` functions.

Please note that this AES-256 bit key also undergoes the OpenSSL process of derived key stretching.

That is:

- `script_password = scrypt( input: password, salt: sha3_256( password ), N: 16384, r: 16, p: 1, dkLen: 32 )`
- `random_salt = crypto.randomBytes( size: 8 )`
- `derived_length = aes_block_size + aes_256_key_size`
- `derived_string = SHA3_KMAC( input: scrypt_password, salt: random_salt, length: derived_length )`
- `derived_iv = derived_string.slice( 0, aes_block_size )`
- `derived_key = derived_string.slice( aes_block_size )`

Following this, the `GCM` encryption process is as follows:

```javascript
/* Convert the database to a string format. */
let database_config_string = JSON.stringify( database_config_object );

/* Pad the message to the AES block size. */
database_config_string = __padMessage( database_config_string, 'PKCS-7', aes_block_size );

/* Create the cipher with derived IV and key. */
let _encrypt = crypto.createCipheriv( 'aes-256-gcm', derived_key, derived_iv );

/* Disable automatic PKCS #7 padding. We do this in-house. */
_encrypt.setAutoPadding( false );

/* Get the cipher text. */
let _ct = _encrypt.update( database_config_string, undefined, 'hex' );
_ct += _encrypt.final('hex');

/* Finally, the authentication tag and the random salt used is prepended to the message. */
_ct = _encrypt.getAuthTag().toString('hex') + random_salt.toString('hex') + _ct;

/* Return the encrypted data as Base64 text. */
return _ct.toString('base64');
```

During decryption, the authentication tag is stripped off as well as the random salt.

The `scrypt` derived password is then used with the one-time salt to derive a `key` and 
    `iv` using the same `SHA3_KMAC` process.

Finally, the authentication tag is assigned to `GCM` and verified which either throws an 
    error if message authentication fails or returns the plaintext message.

**N.B. Only a single authentication tag is produced for the final ciphertext in 
    multi-encryption even though two salts of 64-bits in length are used for key derivation.**

## Key Exchange Process

**DiscordCrypt** uses the Diffie-Hellman exchange algorithm to derive a unique 
    [shared secret](https://en.wikipedia.org/wiki/Shared_secret).

Once both parties post a public key, a shared secret of the algorithm's bit length is derived.

This secret, along with the two [`user salts`](#public-key-format) attached to each 
    message undergoes [key stretching](https://en.wikipedia.org/wiki/Key_stretching) to 
    derive a primary and secondary password via the `Scrypt` hashing algorithm.

The way these keys are produced follows:

* Derive a shared secret using the Diffie-Hellman algorithm.
* Extract both salts attached to each public key message.
* Choose a primary salt by checking which salt is larger than the other.
* Define KMAC parameters used for key derivation as:
    * `discordCrypt-primary-secret` & `discordCrypt-secondary-secret`
* Calculate the primary key as:
    * `PrimaryKey = Base64Encode( KMAC( PrimarySalt, DerivedSecret, 2048, KMAC_PRIMARY_PARAM ) )`
* Calculate the secondary key as:
    * `SecondaryKey = Base64Encode( KMAC( SecondarySalt, DerivedSecret, 2048, KMAC_SECONDARY_PARAM ) )`

These steps generate two keys containing roughly `2000` bits of 
    [entropy](https://en.wikipedia.org/wiki/Entropy_(information_theory)), which is 
    calculated using [Shannon's algorithm](https://en.wiktionary.org/wiki/Shannon_entropy).

Each of these derived keys are converted to Base64 and used in conjunction with symmetric 
    ciphers for message encryption.

# Known Vulnerabilities

While **DiscordCrypt** attempts to be as secure as possible, we place reliance on several 
    things that cannot be changed.

A brief overview of the possible attack mechanisms we **ARE** aware of but unfortunately, 
    cannot fix follows.

* **Bugs In NodeJS's `crypto` Module**

> This is perhaps the biggest caveat we're aware of.
> 
> Since NodeJS's modules are used to provide most of the core 
> functionality of this plugin, we place heavy reliance on its ability to be secure. 
> As such, any potential security 
> flaws affecting this module will directly affect the security of the plugin.
>
> This caveat also includes Discord manually compromising the `crypto` module between releases.

* **Possible Imperfections In The Implementation Of Cipher Padding Schemes** 

> As these padding schemes have been manually implemented and not audited by any security teams, 
> there does exist the possibility that due to this, possible flaws in their 
> implementation may lead to weakened or even broken security, however unlikely this event may be.

* **Vulnerable Storage Of The Password Database**

> BetterDiscord unfortunately allows anyone with console or JavaScript executing permissions to 
> directly view the internal structures of any plugin.
>
> These include access to all variables, methods and configurations used as well as the ability 
> to replace them at will.
>
> As the database is stored in an unencrypted fashion once the master password unlocks it, it is 
> possible for someone to steal your passwords.
>
> While we are fully aware of this, we're generally unable to do much about it. As such, we can 
> only issue security advisories.
>
> These being:
>
> - Don't use plugins that have an "auto-update" mechanism allowing possible drive-by attacks 
    to reveal your database.
> - Change old conversation passwords frequently to avoid compromised passwords from revealing 
    much information.
> - Be very cautious when installing BetterDiscord forks as they might contain malicious code.
> - Always audit the code for any plugin you install for malicious activity.
> - Refrain from installing plugins that you do not trust.

* **Man In The Middle Attacks**

> It is possible to perform a MiTM attack on the encryption methods used in this plugin 
> but **ONLY** if Discord itself has been compromised.
>
> This is due to the fact that the plugin cannot tell that a message was truly sent by you 
> and instead relies on Discord's own implementation of encrypted communication.
>
> As such, during a key exchange, if someone manages to intercept your messages and replace  
> your public key with theirs, they can completely decrypt all further messages between you 
> and your colleagues.
>
> Performing such an attack, while difficult due to the fact that all messages are sent via 
> TLS to Discord's servers **is possible if one controls or forces Discord to 
> compromise** these servers. It however cannot be done by a third party without 
> breaking into or compromising Discord's SSL certificate.
> 
> A rather excellent resource explaining this process can be seen in the video below.
>
> [![https://i.imgur.com/Ti6BzSv.png](https://img.youtube.com/vi/vsXMMT2CqqE/0.jpg)](https://youtube.com/watch?v=vsXMMT2CqqE "MiTM Attacks") 
>
> **N.B. These attacks can only be performed during a key exchange as this is the only time 
> reliance is placed on Discord's own security implementations.**
