============================================
 af_alg Kernel Crypto Interface for OpenSSL
============================================

.. image:: https://img.shields.io/badge/license-openssl-blue.svg?style=flat
   :target: https://github.com/sarnold/af_alg/blob/master/COPYING

.. image:: https://badge.fury.io/gh/sarnold%2Faf_alg.svg?style=flat
   :target: https://badge.fury.io/gh/sarnold%2Faf_alg

.. image:: https://travis-ci.org/sarnold/af_alg.svg?branch=master
   :target: https://travis-ci.org/sarnold/af_alg

.. image:: https://codeclimate.com/github/sarnold/af_alg/badges/gpa.svg
   :target: https://codeclimate.com/github/sarnold/af_alg

.. image:: http://githubbadges.herokuapp.com/sarnold/af_alg/issues.svg?style=flat
   :target: https://github.com/sarnold/af_alg/issues

This is based on RidgeRun's autotools version of the original af_alg project.

Requirements
------------

* linux kernel >= 2.6.38 with crypto modules enabled
* openssl source (development headers)

Build
-----

::

  ./configure
  make

Install
-------

::

  make install

Test it
-------

::

    $ openssl speed -evp aes-128-cbc -engine af_alg -elapsed

Configuration - openssl and kernel config
-----------------------------------------

The algorithms run by af_alg can be configured in the openssl.cnf file
by setting the CIPHERS and DIGEST values. Not setting them will speedup nothing.
The idea is to run algorithms via af_alg which can be accelerated via hardware.
Since there is no documented "query" interface, you'll need to know your kernel
config (mainly the cryptographic and library sections) since the key hardware
drivers are specific to each architecture/machine variant.

In your kernel source tree, do::

    $ sudo make menuconfig

and check/enable the crypto modules and user space modules for your architecture::

    -*- Cryptographic API  --->
        Library routines  --->
    
          --- Cryptographic API
          *** Crypto core or helper ***
    <M>   RSA algorithm
    -*-   Cryptographic algorithm manager
    <M>   Userspace cryptographic algorithm configuration

The basic required crypto/hash modules are::

    {*}   Authenc support
    {*}   CCM support
    {M}   CBC support
    -*-   CTR support
    {M}   CTS support
    {M}   ECB support
    {M}   LRW support
    {M}   PCBC support
    {M}   XTS support
          *** Hash modes ***
    {M}   CMAC support
    -*-   HMAC support
    -*-   CRC32c CRC algorithm
    -*-   CRCT10DIF algorithm
    -*-   GHASH digest algorithm
    {M}   Poly1305 authenticator algorithm
    -*-   MD5 digest algorithm
    -*-   SHA1 digest algorithm
    -*-   SHA224 and SHA256 digest algorithm
    {M}   SHA384 and SHA512 digest algorithms
    -*-   AES cipher algorithms
    {*}   ARC4 cipher algorithm
    {M}   CAST5 (CAST-128) cipher algorithm
    {*}   DES and Triple DES EDE cipher algorithms
          *** Compression ***
    {M}   Deflate compression algorithm
    <*>   LZO compression algorithm
          *** Random Number Generation ***
    -*-   NIST SP800-90A DRBG  --->
    -*-   Jitterentropy Non-Deterministic Random Number Generator
    [*]   Hardware crypto devices  --->
    -*-   Asymmetric (public-key cryptographic) key type  --->

The library section is much shorter; if not selected, select the BCJ filter
decoder that matches your hardware::

    {M} CRC32c (Castagnoli, et al) Cyclic Redundancy-Check
    <*> XZ decompression support
    [*]   x86 BCJ filter decoder
    [ ]   PowerPC BCJ filter decoder
    [ ]   IA-64 BCJ filter decoder
    [ ]   ARM BCJ filter decoder
    [ ]   ARM-Thumb BCJ filter decoder
    [ ]   SPARC BCJ filter decoder


In /etc/ssl/openssl.cnf

::

  openssl_conf = openssl_def
  
  [openssl_def]
  engines = openssl_engines
  
  [openssl_engines]
  af_alg = af_alg_engine
  
  [af_alg_engine]
  default_algorithms = ALL
  CIPHERS=aes-128-cbc aes-192-cbc aes-256-cbc des-cbc des-ede3-cbc
  DIGESTS=md4 md5 sha1 sha224 sha256 sha512

This will enforce loading the af_alg OpenSSL dynamic engine by default, so it
can be used by OpenSSH.  Starting with OpenSSH 5.4p1 OpenSSH honors the openssl
config and will use your default engines specified.

Required kernel modules
-----------------------

Make sure you have at least::

    algif_hash             12943  0 
    algif_skcipher         17369  0 
    af_alg                 14686  2 algif_hash,algif_skcipher

in your lsmod output.

If you can't load the modules, check the kernel config options again.  Grep is
your friend here::

    $ grep CRYPTO_USER_API /usr/src/linux/.config

    CONFIG_CRYPTO_USER_API=m
    CONFIG_CRYPTO_USER_API_HASH=m
    CONFIG_CRYPTO_USER_API_SKCIPHER=m
    CONFIG_CRYPTO_USER_API_RNG=m
    CONFIG_CRYPTO_USER_API_AEAD=m

and make sure the above modules are in your kernel config.

Performance
-----------

If you have hardware crypto support for large block sizes, AF_ALG is supposed
to increase performance; for small block sizes, the overhead introduced by
AF_ALG may slow things down.  In case you are looking for better performance,
you might need a dedicated hardware crypto device.  Cryptodev is another option,
however, cryptodev is also somewhat slower for smaller block sizes, but should
provide a significant boost for 8192 size blocks.

::

    engine "af_alg"
    type             16 bytes     64 bytes    256 bytes   1024 bytes   8192 bytes
    aes-128-cbc       7952.93k    33862.40k   141754.51k   640204.80k  2575564.80k
    
    engine "cryptodev"
    aes-128-cbc       5654.96k    17000.96k   141747.20k   384430.08k  2564915.20k
    
    engine "builtin" (Cavium Octeon modules)
    aes-128-cbc       9700.32k    86694.40k    91764.36k   646519.47k  2578841.60k

.. Note::
   The above numbers were generated on an EdgeRouter Lite mips64 system using a mainline kernel.
   
   Linux edgerouter 4.5.1 #7 SMP PREEMPT Thu Apr 21 12:37:02 PDT 2016 mips64 Cavium Octeon+ V0.1 UBNT_E100 (CN5020p1.1-500-SCP) GNU/Linux


Debugging
---------

OpenSSL ships evp_test, which can be used to verify things work.
A patch on OpenSSL is required to force evp_test using the config.

::

    diff --git a/crypto/evp/evp_test.c b/crypto/evp/evp_test.c
    index ad36b84..d40c461 100644
    --- a/crypto/evp/evp_test.c
    +++ b/crypto/evp/evp_test.c
    @@ -532,8 +532,8 @@ int main(int argc,char **argv)
         /* Load all compiled-in ENGINEs */
         ENGINE_load_builtin_engines();
     #endif
    -#if 0
    -    OPENSSL_config();
    +#if 1
    +    OPENSSL_config(NULL);
     #endif
     #ifndef OPENSSL_NO_ENGINE
         /* Register all available ENGINE implementations of ciphers and digests.

Create a config /tmp/af_alg.cnf with mentioned modifications to force using the
engine::

  export OPENSSL_CONF=/tmp/af_alg.cnf
  openssl/test$ ./evp_test evptests.txt

It will fail if the computed results do not match the expected results.
Compiling the engine with::

  make CFLAGS=-DDEBUG clean all

may help as well.

Other ways
----------

cconf can be used to modify the crypto priorities on kernels >= 3.2


References
----------

  * https://events.linuxfoundation.org/sites/events/files/slides/lcj-2014-crypto-user.pdf
  * http://article.gmane.org/gmane.linux.kernel.cryptoapi/5292
  * http://article.gmane.org/gmane.linux.kernel.cryptoapi/5296
  * https://bugzilla.mindrot.org/show_bug.cgi?id=1707
  * http://thread.gmane.org/gmane.linux.kernel.cryptoapi/6045
  * http://sourceforge.net/projects/crconf/
 
Authors
-------

  * Markus Koetter
  * Carsten Behling <carsten.behling@ridgerun.com>
  * Stephen Arnold <stephen.arnold42@gmail.com>

