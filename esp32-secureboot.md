# Understanding Secure Boot ESP32 series (Part 1)

This article is part of a series on the Security Features of the Espressif ESP32 microcontroller series. This includes MCUs based on the Xtensa Instruction Set (e.g. ESP32, ESP32-S3), as well as MCUs based on the RISC-V Instruction Set (e.g. ESP32-C3). This is the first article of the series, which will cover:
* Secure Boot V1 (AES Based Secure Boot) (this article)
* Secure Boot V2 (RSA Based Secure Boot)
* Flash encryption (AES-XTS and legacy)
* AES, SHA, RSA, DS and HMAC accelerators
* World Controller to allow isolated execution environments

## Introduction

[ESP32 SoCs](https://www.espressif.com/en/products/socs) are very popular System On a Chip (SoC) microcontrollers made by the company [Espressif](https://espressif.com/). The CPU cores used in these are RISC-V or Xtensa cores, depending on the model. They are high-performance, low-cost and embed an impressive amount of hardware peripherals. While they are well known for their wireless capabilities (Wi-Fi, Bluetooth), they have an impressive amount of security features and peripherals built in hardware. This includes: Secure Boot, Flash Encryption, a true Random Number Generator (RNG), Hardware Accelerated SHA, AES, HMAC, and RSA, and a World Controller to enable the implementation of a trusted execution environment (TEE). For these reasons, these chips are now found in several consumer electronics products, as well as in IoT and industrial applications. 
 
In this article, I will discuss the Secure Boot feature of the ESP32 in version 1. Please note that for new designs, it is recommended to use Secure Boot V2, which I will cover in a future article. There are however several reasons to study Secure Boot V1: MCUs supporting only Secure Boot V1 can't be upgraded to Secure Boot V2. That is a new chip revision, a new silicon. This means that using V2 is only for new designs. Also, to understand the technical choices that Espressif made with V2, learning the weaknesses of V1 is essential. In any case, looking at this implementation is really interesting for anyone willing to learn about security in embedded systems.

## What is Secure Boot

With my threat modeler hat, which I never remove when I do security work, I will first clarify what security threats Secure Boot is attempting to mitigate, then I'll provide a high level explanation on how it works. Finally in this article, we'll review the important implementation details to account for when using Secure Boot on these SoCs.

### Threat Model

When one implements a security feature, it should be crystal clear, what security threat this feature attempts to address. I generally recommend that every security requirement is documented and includes an explicit mapping to the description of the threat this security control is attempting to mitigate. Otherwise, and it happens more often than one may think, there is a lack of clarity into why a specific security feature is implemented. This can result in a false sense of security. And this, in turn, can have severe consequences for the security of the device.

Secure Boot on embedded systems is implemented as a control to ensure that only the firmware created by the device's manufacturer can be run on the device.

Let's look at a branch of an attack tree on embedded systems, relevant to secure boot:
>* **T1** Attacker modifies firmware or runs custom firmware on the device in order to extract sensitive information such as personal information, or credentials and cryptographic keys (stored in other memory locations or peripherals), or to unlock functions on a device, leading to loss or revenue for the device manufacturer. `(AND)`
>    * **T1.1** By creating valid modified firmware image `(OR)`
>        * **T1.1.1** By exploiting weaknesses in, or absence of, Secure Boot on the target device
>        * **T1.1.2** By using compromised signing key material
>        * **T1.1.3** By exploiting vulnerabilities on an online firmware signing service
>    * **T1.2** By uploading modified firmware on the target device `(OR)`
>        * **T1.2.1** By exploiting weaknesses in over the air firmware update (OTA) feature of the device `(OR)`
>            * **T1.2.1.1** Exploiting lack of secure channel with the firmware server (e.g. not using TLS)
>            * **T1.2.1.2** Exploiting improper certificate validation during https download on firmware server
>        * **T1.2.2** By uploading a custom second stage bootloader and application switching the Boot ROM in serial bootloader mode.
>        * **T1.2.3** By uploading a custom second stage bootloader and application by directly updating content on external flash chip (e.g. directly writing to the chip by SPI),
>        * **T1.2.4** By uploading a custom second stage bootloader and application by JTAG Direct flashing.

Once you have established a list of threats you're concerned about, you need to identify the different mitigation necessary. Most of the time, and to support defense in depth, several layers of security measures are necessary. For instance mitigating  **T1.2.4** on ESP SoCs would include **M1** *Secure Boot*, **M2** *Flash Encryption* to prevent TOCTOU attacks on Secure Boot, **M3** *Secure key management* for firmware provisioning, **M4** *Disabling JTAG* by burning relevant eFuse, and **M5** *not exposing JTAG headers on the PCB*.

This article focuses merely on **M1**, but I hope the explanation above illustrates the importance of threat modeling for embedded device security, and the proper selection of security controls necessary.

Also, I should note that a common misconception is to think that Secure Boot mitigates the threats of an attacker accessing and retrieving the firmware on the devices. Secure Boot doesn't protect against these attacks, which enable the attacker to analyze and learn from the vendor's firmware (identify exploitable vulnerabilities in the firmware).These attacks can allow retrieving secrets stored in firmware (which should be prevented). Firmware/Flash encryption are mechanisms put in place to mitigate these categories of threats, and will be covered in future articles.
Finally, it is also important to note that Secure Boot doesn't guarantee your code is secure, and vulnerabilities free. It merely attempts to provide guarantees that the code running on the embedded system is only the code, which was provided by the vendor/manufacturer.

## Secure boot on ESP32 family

Secure boot means the code running on the device is trusted. It provides confidence that the code executed has not been tampered with (Integrity) and has been issued by a trusted entity (Authenticity), the device's manufacturer for instance. 
This is achieved by building a chain of trust. On ESP32, typically, this chain is composed of:
1. The Boot ROM (*First Stage Bootloader*)
2. The bootloader in Flash (*Second Stage Bootloader*)
3. The firmware application and its partition table.

Each element of the chain has mechanisms to verify the integrity and authenticity of the next element (it trusts it). Once an element verifies successfully the next element, it transfers control to it. The first element, the Boot ROM (or First stage bootloader) is the root of trust (RoT). The Boot ROM is inherently trusted because it is added during the manufacturing process of the chip, and stored in read-only-memory (ROM). This Boot ROM can't be modified or erased. Please note it doesn't mean that since it's trusted it can't contain vulnerabilities. It only means that we trust that the code on the Boot ROM was provided by Espressif. Vulnerabilities are regularly found in Boot ROMS with various microcontrollers. Since the Boot ROM can't be updated, new revisions of chips need to be made to address vulnerabilities. Existing devices with the vulnerable chip can't be updated.


```
        ┌───────── First Stage bootloader in hardware
        │          Is the Root of Trust (RoT) added during the manufacturing process of the chip
┌───────────────────┐
│ Boot ROM          │
│                   │
│ (First Stage      │
│  bootloader)      │ ─────┐
└───────────────────┘      │ (1)
                           │ Verifies then
                           │ Pass control to
                           │
                           │
┌───────────────────┐      │
│ Bootloader Flash  │ ◄────┘
│                   │
│ (Second Stage     │
│  bootloader)      │ ─────┐
└───────────────────┘      │ (2)
                           │ Verifies then
                           │ Pass control to
                           │
                           │
┌───────────────────┐      │
│ Application       │ ◄────┘
│                   │
│                   │
│                   │
└───────────────────┘
```
### First stage: verification of the Second Stage bootloader (1)

The Boot ROM (AKA First Stage Bootloader) is the first code that runs when the ESP32 is powered on or reset. It has the highest level of access, including the ability to read all eFuses (even if they are read protected). The Boot ROM is responsible for initializing the system, loading the bootloader flash, and verifying it. Since the Boot ROM is “hard-wired” into the chip during the manufacturing process, it is considered the root of trust (RoT) in the ESP32’s secure boot process.

The Boot ROM verifies a digest of the Bootloader stored in Flash memory (AKA Second Stage Bootloader) before passing control to it. This is to ensure the Second Stage Bootloader hasn't been tampered with, or replaced by one, which was not issued by the device manufacturer. There are interesting characteristics of this verification regarding how it's done with this version of the secure boot on ESP32.

One significant aspect is that the verification mechanism doesn't rely on public key cryptography. (Where a private key is used to sign a digest, and a public key is used to verify the signature.) Digital signature would be typically done with algorithms such as RSA, DSA or ECDSA. In the case of Secure Boot v1, a unique secret key is used to form what Espressif calls a digest. I think it's more suitable to call it a Message Authentication Code (MAC). It is using symmetric cryptography instead of public key cryptography. This means that the same secret is used for verifying the Bootloader in Flash, but also for "signing" new ones. Things often can go wrong when Messages Authentication Codes are used in places where digital signatures should be used (using a symmetric key, instead of a public key for verification). I'll explain later in this article why this choice has important ramifications in the case of Secure Boot v1.

There are more details to it, related to hardware restrictions and not cryptographic choices. (I won't cover these, but provide at the end of this article the Python code showing the implementation used to generate the authentication code.) Here is a simplified view of how this authentication code is generated. All of this is done in hardware, the AES key is stored in eFuse (block2) and is not accessible by software.

{{< highlight pascal  >}}
SHA-512( AES-256-ECB( IV || Plaintext_Bootloader_Image ))
{{< / highlight >}}

Output digest is 192 bytes of data: the 128-byte IV, followed by the 64-byte SHA-512 digest. It is flashed at offset `0x0` in the flash (before the bootloader, which is flashed at `0x1000`).
So that's interesting. It fulfils a similar objective as a Message Authentication Code (MAC). This is a message authentication code constructed from the combination of a block cipher and a secure hash algorithm but it's not using a known construction such as HMAC or CBC-MAC. 

### Second Stage: verification of the application (2)

Once the Second Stage Bootloader is verified, it's given the control. In order to continue the secure boot chain, this second bootloader, stored in flash, needs to verify the application image (and its partition table). For this verification, Espressif chose to use a digital signature scheme, using Deterministic Elliptic Curve Digital Signature Algorithm (Deterministic ECDSA).

I don't know why Espressif chose a deterministic scheme for ECDSA, but I'll take the liberty to speculate on the reason:
* ECDSA needs a cryptographically secure source of randomness. While the ESP32 SoC contains a [hardware RNG](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/random.html), only non-RF entropy sources are enabled during the Second Stage Bootloader stage.
* Several successful attacks on ECDSA are related to issues with the generation of the random number *k*. For instance: [Recovery of private key to sign games for PlayStation 3 console](https://en.wikipedia.org/wiki/PlayStation_3_homebrew#Private_key_compromised), or [Android bug batters Bitcoin wallets](https://www.theregister.com/2013/08/12/android_bug_batters_bitcoin_wallets/).

Here are some details from Espressif on how the signature is generated:
* Curve is NIST256p. OpenSSL calls this curve prime256v1, and it is also sometimes called secp256r1.
* The hash function is SHA256.
* The key format used for storage is PEM.
    * In the bootloader, the public key for signature verification is flashed as 64 raw bytes.
* Image signature is 68 bytes: a 4-byte version word (currently zero), followed by 64 bytes of signature data. These 68 bytes are appended to an app image or partition table data.


### Weaknesses of Secure Boot V1

The main weakness of Secure Boot V1 on the ESP32 resides in the First Stage Bootloader (Boot ROM). The issue is the scheme being used to verify the authenticity and integrity of the next bootloader is not using public key cryptography. The Boot ROM relies on a shared key to generate a digest. (Which I prefer calling it a Message Authentication Code - MAC.) It uses **the same secret value (AES Key) for both verifying the Bootloader flash digest, and for generating new ones**. This means that if the AES key is retrieved, new custom firmware can be "signed" (signature is not the correct term here).

This is actually a problem I often see when doing code review of authentication algorithms. While handling a unique symmetric key makes key management simpler (versus managing public key infrastructures), it assumes that the component verifying authenticity of a message, can also bear the roles of asserting authenticity, and in many cases I reviewed, that's not desirable. What does it mean in the case of Secure Boot v1? Well it means that if the AES digest verification key is obtained, any new firmware can be signed with this key. This breaks the security objective of secure boot as an attacker can upload a new version of the bootloader flash, which will remove application signature verification.

In the case of ESP32 chips, especially the silicons before revision v3.0, the problem is exacerbated by [successful fault injection attacks](https://limitedresults.com/2019/11/pwn-the-esp32-forever-flash-encryption-and-sec-boot-keys-extraction/), using voltage glitching and allowing retrieval of the AES key. See also [CVE-2019-17391](https://www.espressif.com/en/news/Security_Advisory_Concerning_Fault_Injection_and_eFuse_Protections%20) and [ESP32 Fault Injection Vulnerability - Impact Analysis](https://www.espressif.com/en/news/ESP32_FIA_Analysis).

#### Well it could be worse...

One important aspect to consider is that if the AES key is unique per device, this key recovery attack is not scalable to multiple devices. This means that in order to run custom firmware on multiple devices, physical access to each device is necessary, and the fault injection attack needs to be performed on each device, too. This is very different from an attack like the PlayStation 3 private key access mentioned previously. Now, this could be still really bad depending on your use-case (e.g. a device deployed in public space, or a device containing secrets shared across multiple products). Therefore you should always evaluate the impact for your own application.

#### But you can make it worse...

Ok, now this is where you can really shoot yourself in the foot. There are two secure bootloader modes: **One-time Flash** and **Reflashable**.

With the **One-time Flash mode**, the first time the device boots it generates a unique AES key with the chip's RNG, burns it in eFuses, calculates the digest and stores it in flash. In this mode it's not possible to flash another bootloader, because there is no way to generate a new digest for this new bootloader.

If you want to have secure boot enabled and still have the possibility to update your Second Stage bootloader in flash, then you need the **Reflashable mode**. And this is where things can get really ugly. While there are legitimate needs to use secure boot and reflashable boot loader, you still want a unique AES key per device. However, the default behaviour of the ESP-IDF build process in this mode, is to derive the AES key, using SHA-256 with the ECDSA app signing key. And now this is bad. Because generally the ECDSA signing key is the same for all devices. Therefore in this scenario, you're using the same AES key across all your devices, and you make your key recovery attack scalable, bad times.

Finally I want to talk about another configuration option, which is **Signed App Verification Without Hardware Secure Boot**. It could be alright if you understand what threats this feature is addressing. In this mode, the Root of Trust (RoT) is the Second Stage Bootloader in Flash, not the Boot ROM. As the code for this RoT is mutable (in flash), your devices will still be vulnerable to several threats, especially all threats with physical access to the device. (E.g. **T1.2.2**, **T1.2.3** and **T1.2.4** in the examples previously described.) There are however use cases, where using this option can make sense depending on your threat model.

### Recommendations

Obviously the best recommendation is to use Secure Boot V2 if you can (if you use a version of silicon, which supports it). However, if you can't, there are a few things, which can be done to improve security of Secure Boot V1:
1. **Use a unique AES key per device**. This is the main recommendation. Always do that with V1.
2. **Let the devices generate the AES key for you**. With this way, the key is never exposed to any system or application other than the ESP hardware. You can't lose what you don't have, one can't steal what you don't have. It's a golden rule in security, if you don't really need it, don't handle it.
3. **Use flash encryption**. If secure boot is used without Flash Encryption, your device is vulnerable to a time-of-check to time-of-use attack (TOCTOU), where flash contents is changed just after image verification by the bootloader.
4. **Disable JTAG and ROM BASIC interpreter**. For defense in depth, you want to disable the different entry points used to upload new firmware, where possible.

### Implementation reference of Flash Bootloader digest (MAC) generation

This code is an extract of the [espsecure.py](https://github.com/espressif/esptool/blob/3a82d7a2d31f509038a5947ae73c3e488be5d664/espsecure/__init__.py#L89-L151) tool provided by Espressif, it is used when it is desired that the Second Stage Bootloader is not signed in Hardware. This gives the right level of details in the implementation. 

The creation of the digest is highlighted (lines 121 to 140):   

{{< highlight python2 "linenostart=89, hl_lines=33-52" >}}
def digest_secure_bootloader(args):
    """Calculate the digest of a bootloader image, in the same way the hardware
    secure boot engine would do so. Can be used with a pre-loaded key to update a
    secure bootloader."""
    _check_output_is_not_input(args.keyfile, args.output)
    _check_output_is_not_input(args.image, args.output)
    _check_output_is_not_input(args.iv, args.output)
    if args.iv is not None:
        print("WARNING: --iv argument is for TESTING PURPOSES ONLY")
        iv = args.iv.read(128)
    else:
        iv = os.urandom(128)
    plaintext_image = args.image.read()
    args.image.seek(0)

    # secure boot engine reads in 128 byte blocks (ie SHA512 block
    # size), but also doesn't look for any appended SHA-256 digest
    fw_image = esptool.bin_image.ESP32FirmwareImage(args.image)
    if fw_image.append_digest:
        if len(plaintext_image) % 128 <= 32:
            # ROM bootloader will read to the end of the 128 byte block, but not
            # to the end of the SHA-256 digest at the end
            new_len = len(plaintext_image) - (len(plaintext_image) % 128)
            plaintext_image = plaintext_image[:new_len]

    # if image isn't 128 byte multiple then pad with 0xFF (ie unwritten flash)
    # as this is what the secure boot engine will see
    if len(plaintext_image) % 128 != 0:
        plaintext_image += b"\xFF" * (128 - (len(plaintext_image) % 128))

    plaintext = iv + plaintext_image

    # Secure Boot digest algorithm in hardware uses AES256 ECB to
    # produce a ciphertext, then feeds output through SHA-512 to
    # produce the digest. Each block in/out of ECB is reordered
    # (due to hardware quirks not for security.)

    key = _load_hardware_key(args.keyfile)
    backend = default_backend()
    cipher = Cipher(algorithms.AES(key), modes.ECB(), backend=backend)
    encryptor = cipher.encryptor()
    digest = hashlib.sha512()

    for block in get_chunks(plaintext, 16):
        block = block[::-1]  # reverse each input block

        cipher_block = encryptor.update(block)
        # reverse and then byte swap each word in the output block
        cipher_block = cipher_block[::-1]
        for block in get_chunks(cipher_block, 4):
            # Python hashlib can build each SHA block internally
            digest.update(block[::-1])

    if args.output is None:
        args.output = os.path.splitext(args.image.name)[0] + "-digest-0x0000.bin"
    with open(args.output, "wb") as f:
        f.write(iv)
        digest = digest.digest()
        for word in get_chunks(digest, 4):
            f.write(word[::-1])  # swap word order in the result
        f.write(b"\xFF" * (0x1000 - f.tell()))  # pad to 0x1000
        f.write(plaintext_image)
    print("digest+image written to %s" % args.output)
{{< / highlight >}}

### Implementation reference of image signature verification

This code is an extract of the [espsecure.py](https://github.com/espressif/esptool/blob/3a82d7a2d31f509038a5947ae73c3e488be5d664/espsecure/__init__.py#L724-L761) tool provided by Espressif, it show how the image signature is verified by the Second Stage Bootloader. Obviously on the device, this is in the bootloader code in C. (Look in ESP-IDF at `components/bootloader_support/src` or [here](https://github.com/espressif/esp-idf/tree/master/components/bootloader_support/src))

{{< highlight python2 "linenostart=724">}}
def verify_signature_v1(args):
    """Verify a previously signed binary image, using the ECDSA public key"""
    key_data = args.keyfile.read()
    if b"-BEGIN EC PRIVATE KEY" in key_data:
        sk = ecdsa.SigningKey.from_pem(key_data)
        vk = sk.get_verifying_key()
    elif b"-BEGIN PUBLIC KEY" in key_data:
        vk = ecdsa.VerifyingKey.from_pem(key_data)
    elif len(key_data) == 64:
        vk = ecdsa.VerifyingKey.from_string(key_data, curve=ecdsa.NIST256p)
    else:
        raise esptool.FatalError(
            "Verification key does not appear to be an EC key in PEM format "
            "or binary EC public key data. Unsupported"
        )

    if vk.curve != ecdsa.NIST256p:
        raise esptool.FatalError(
            "Public key uses incorrect curve. ESP32 Secure Boot only supports "
            "NIST256p (openssl calls this curve 'prime256v1"
        )

    binary_content = args.datafile.read()
    data = binary_content[0:-68]
    sig_version, signature = struct.unpack("I64s", binary_content[-68:])
    if sig_version != 0:
        raise esptool.FatalError(
            "Signature block has version %d. This version of espsecure "
            "only supports version 0." % sig_version
        )
    print("Verifying %d bytes of data" % len(data))
    try:
        if vk.verify(signature, data, hashlib.sha256):
            print("Signature is valid")
        else:
            raise esptool.FatalError("Signature is not valid")
    except ecdsa.keys.BadSignatureError:
        raise esptool.FatalError("Signature is not valid")
{{< / highlight >}}

## That's all folks!
That's it for a long blog article, in the next one I will focus on Secure Boot V2.
