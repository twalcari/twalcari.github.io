---
layout: post
title: "Signing Java .jar Files with a Hardware Token in Linux"
comments: true
date: "2015-07-31"
---

Signing your Java code with an *EV code signing certificate* is significantly harder
than with a regular code signing certificate. The biggest difference is that the
EV code signing certificate is stored on a hardware dongle.

We received an Aladdin eToken Pro 72k from our certificate provider Digicert.
After activating and initializing this hardware token, the following steps were
needed to get code signing working on Ubuntu 14.04 .


## Installing support for Hardware tokens on Linux

1. Install the necessary packages to support hardware tokens:

    # apt-get install openct libpcsclite1 pcscd pcsc-tools

2. Install the drivers of your hardware token.

These should be provided to you by your certificate provider. If you're using the
Aladdin eToken Pro 72k, you can also download the "Safenet Authentication client",
which contains all necessary drivers from <http://www.zaba.hr/home/wps/wcm/connect/zaba_hr/zabautils/onlineusluge/e-zaba+poslovno+bankarstvo/ezaba_na_linuxu>.

   # dpkg -i SafenetAuthenticationClient-9.0.43-0_amd64.deb

After completing these steps, you should be able to read out the hardware token
as follows:

    # pcsc_scan
    PC/SC device scanner
    V 1.4.22 (c) 2001-2011, Ludovic Rousseau <ludovic.rousseau@free.fr>
    Compiled with PC/SC lite version: 1.8.10
    Using reader plug'n play mechanism
    Scanning present readers...
    0: AKS ifdh [Main Interface] 00 00

    Thu Jul 30 16:43:29 2015
    Reader 0: AKS ifdh [Main Interface] 00 00
    Card state: Card inserted,
    ATR: 3B D5 18 00 81 31 FE 7D 80 73 C8 21 10 F4

    ATR: 3B D5 18 00 81 31 FE 7D 80 73 C8 21 10 F4
    + TS = 3B --> Direct Convention
    + T0 = D5, Y(1): 1101, K: 5 (historical bytes)
    TA(1) = 18 --> Fi=372, Di=12, 31 cycles/ETU
    129032 bits/s at 4 MHz, fMax for Fi = 5 MHz => 161290 bits/s
    TC(1) = 00 --> Extra guard time: 0
    TD(1) = 81 --> Y(i+1) = 1000, Protocol T = 1
    -----
    TD(2) = 31 --> Y(i+1) = 0011, Protocol T = 1
    -----
    TA(3) = FE --> IFSC: 254
    TB(3) = 7D --> Block Waiting Integer: 7 - Character Waiting Integer: 13
    + Historical bytes: XX XX XX XX XX
    Category indicator byte: 80 (compact TLV data object)
    Tag: 7, len: 3 (card capabilities)
      Selection methods: C8
        - DF selection by full DF name
        - DF selection by partial DF name
        - Implicit DF selection
      Data coding byte: 21
        - Behaviour of write functions: proprietary
        - Value 'FF' for the first byte of BER-TLV tag fields: invalid
        - Data unit in quartets: 2
      Command chaining, length fields and logical channels: 10
        - Logical channel number assignment: by the card
        - Maximum number of logical channels: 1
    + TCK = F4 (correct checksum)

    Possibly identified card (using /usr/share/pcsc/smartcard_list.txt):
    3B D5 18 00 81 31 FE 7D 80 73 C8 21 10 F4
    Bank of Lithuania Identification card
    Aladdin PRO/Java card http://www.aladdin-rd.ru/catalog/etoken/java/

Sensitive values from this output have been redacted.

## Configuring Java on Linux to use a hardware token

Hardware tokens are contacted via the "sun.security.pkcs11.SunPKCS11" provider.
This provider needs a small configuration file to work:

{% gist twalcari/b002d459d45b22a329ef eToken.cfg %}

Note that the location and name of the library can differ depending on your Linux
distribution and type of hardware token.

You should now be able to fetch the certificates which are present on the token
with the Java *keytool*-tool:

    $ keytool -list \
    -keystore NONE \
    -storetype PKCS11 \
    -providerclass sun.security.pkcs11.SunPKCS11 \
    -providerArg eToken.cfg

You should get the following output:

    Enter keystore password: ***********************

    Keystore type: PKCS11
    Keystore provider: SunPKCS11-eToken

    Your keystore contains 1 entry

    YOUR-CERTIFICATE-NAME, PrivateKeyEntry,
    Certificate fingerprint (SHA1): XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX

Signing a jar is done with the *jarsigner*-tool:

   $ jarsigner -verbose \
   -tsa http://timestamp.digicert.com \
   -keystore NONE \
   -storetype PKCS11 \
   -providerClass sun.security.pkcs11.SunPKCS11 \
   -providerArg eToken.cfg \
   "/path/to/file.jar" "YOUR-CERTIFICATE-NAME"
