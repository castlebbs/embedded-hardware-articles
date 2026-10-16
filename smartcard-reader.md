
## CP8 smartcard reader: exploring the carte bleue (credit card)

_By castlebbs on Tuesday 13 February 2007, 14:57_

> I bought a used smartcard reader for 1 euro in a store in Toulouse. This rather old reader works perfectly under Linux; I decided to write this page to [...]
> The author cannot in any case be held responsible for any prejudice or damage of any kind that may result from the use of his explanations or programs.


### Reader connection

#### Power supply

The reader must be powered with 12 V DC. I did not measure its consumption.
I used a 12 V / 1200 mA transformer (there is probably some margin :-) ). The connector of my power supply is a 5.5 mm diameter power jack, + on the center.

#### Serial cable

This was my first problem because I did not know the pinout of the reader. The reader's serial connector is an RJ45 jack. I dismantled the reader and identified the component **LT1181[...]**
From the datasheet and an inspection of the routing, I used the following pinout:

> Pin 1 is on the left of the female RJ45 connector, clip down.
>
> - `5 (rxd)` connected to `3 (txd)` of the DB9 on the PC side
> - `3 (txd)` connected to `2 (rxd)` of the DB9 on the PC side
> - `1 (ground)` connected to `5 (ground)` of the DB9 on the PC side
>
> The `T1out` pin of the serial driver is connected to pin `7` of the RJ45; I do not know what it is used for and I did not wire it.

Consult the **DB9 and DB25 pinouts** to fabricate the PC-side connector.

### Operation under Linux

After research on the Internet, this reader communicates with the computer using the **TLP 224** protocol.
A driver that allows sending ISO commands to the card exists and works: _http://www.gemplus.com/techno/tlp_drivers/_

This driver is unfortunately not PC/SC compatible, which means it is not possible to use applications written for the **MUSCLE** framework.

#### Installation

1. Download `libtlp_1.0.5.tar.gz`
2. In the `src` directory:

   ```bash
   make && make install
   ```

   (modify the Makefile if you want to change the install prefix)

3. Create a link `/dev/screader` for example to the serial port COM1:

   ````bash
   ln -s /dev/ttyS0 /dev/screader
   ````

#### Test

Insert a card into the reader, send a **reset** to the card:

```text
[david@castlebbs tools]$ scsend R
R
C0651125080038046C9000
```

The card has been initialized, the reader's orange LED lights up. To turn the card off:

```text
[david@castlebbs tools]$ scsend P
P
```

The LED is now off.

### Operation under Windows

I never tested the reader under Windows. Information can be found on the fr.comp.carte-a-puce forum, including a relevant thread.
I have read that people use a **program under Windows**.

### Practical use: exploring a carte bleue

This section explains how to read a carte bleue as described by Laurent PELE, but with the Bull CP8 reader and under Linux.
**For educational purposes only.** Any misuse will give rise to prosecution according to article L 335-3 paragraph 2 of the Intellectual Property Code.

#### Reading the free area of the chip

First read of the free area of the chip:

```text
[david@castlebbs tools]$ scsend "O BC B0 09 E0 20"
O BC B0 09 E0 20
3FE52002084D00250935FF671C2B9F55000000000000000000000000000000009000
9000: Normal processing
```

#### Addressing zones

Reading the ADL addressing zone:

```text
[david@castlebbs tools]$ scsend "O BC B0 09 C8 08"
O BC B0 09 C8 08
1FD10BEB0BEB0B7F9000
9000: Normal processing
```

Note that on my test card, the first 16 bits `1FD1` are not the same as on Laurent PELE's page, and also do not match `1FF4` as written in the section. The card I tested was issued in November 2001 — this means that the addressing has been modified.
Here is the table of address zones for my card:

| Zone name | Definition address | Address definition bytes | Address zone result |
| -------- | ------------------ | ------------------------- | ----------------------- |
| ADL      | 09C8               | 1FD1                      | 07F0                    |
| ADT      | 09CC               | 0BEB                      | 02F8                    |
| ADC      | 09D0               | 0BEB                      | 02F8                    |
| ADM      | 09D4               | 0B7F                      | 02C0                    |
| AD2      | 09D8               | 0B10                      | 0B10                    |
| ADS      | 09DC               | 08D9                      | 0230                    |
| AD1      | 09E8               | 084D                      | 0210                    |

Bit manipulation is quite simple: to obtain the address of the ADL zone, you just perform the following operation:

```text
(0x1FD1 & 0xFFE0) >> 2
```

Script used to generate this table:

```sh
#!/bin/sh

# Calculation of address zones. 2004 - David ROBERT
ADR=" 09C8
      09CC
      09D0
      09D4
      09D8
      09DC
      09E8"
echo "$ADR" | while read ADD
do
    scsend "O BC B0 $ADD 02" | gawk --non-decimal-data 'NR==2 {
      val="0x" substr($_,0,4);
      printf ("0x%x, 0x%x\n", val, rshift(and(val, 0xFFE0),2))
    }'
done
```

#### Chip provider identifier zone

On my card, the provider identifier zone 02 is stored at address `08 58`. I can read it with:

```text
[david@castlebbs tools]$ scsend "O BC B0 08 58 40"
O BC B0 08 58 40
------------------------------------------------------ …
9000: Normal processing
```

I did not publish the identifier zone of my card online. I wrote a small Python program to decode the information in this zone:

```python
#!/usr/bin/env python
# decode.py – Decode the CB identifier zone – 2004 – David ROBERT

import sys

try:
    val = sys.argv[1]
except:
    print "usage : decode.py 2E0238F1300497..."
    sys.exit(1)

pointeur = 0
newval = ""
for quarte in val[8:]:
    if pointeur % 8 != 0:
        newval = newval + quarte
    pointeur = pointeur + 1

print "Card number : " + newval[2:18]
print "Usage code : " + newval[21:24]
print "Issue date : %s/%s (year/month)" % (newval[24:26], newval[26:28])
print "Language code : %s (250 = French)" % newval[28:31]
print "Expiration date : %s/%s (year/month)" % (newval[31:33], newval[33:35])
print "ISO numeric code : %s  (250 = francs)" % (newval[35:38])
print "Exponent : %s" % newval[38:39]

nom = newval[42:100]
stringnom = ""
compteur = 0
while compteur < len(nom):
    bytevalue = int("%s%s" % (nom[compteur], nom[compteur+1]), 16)
    stringnom = stringnom + "%c" % bytevalue
    compteur = compteur + 2

print "Cardholder name : " + stringnom
```

**Using the program:**

```text
[david@castlebbs tools]$ /home/david/decode.py 2E0238F1300497...A9000
Card number : 497-------------
Usage code : 101
Issue date : 01/11 (year/month)
Language code : 250 (250 = French)
Expiration date : 03/12 (year/month)
ISO numeric code : 250  (250 = francs)
Exponent : 5
Cardholder name : MR DAVID ROBERT
```

#### Authentication zone

Reading the authentication zone (provider 3), at ADL address `07 F0` on my card:

```text
[david@castlebbs tools]$ scsend "O BC B0 07 F0 34"
O BC B0 07 F0 34
-------------------------------------------------------
9000: Normal processing
```

The "---" are of course replaced by the authentication value.

##### Verifying the chip's authentication value

I had to remove the 4 bytes `9000` at the end and extract the nibbles of `3` every 4 bytes.
The following Python code prepares the value (`val` contains the authentication value as a string):

```python
for quarte in val[:-4]:
    if pointeur % 8 != 0:
        newval = newval + quarte
    pointeur = pointeur + 1
print newval
```

For calculations, a simple Python interpreter is enough:

```python
produit = 213598703592091008...
authent = 0x000074..
"Result : %x" % (authent**3 % produit)
```

#### Reading a PIN-protected zone

Take the PIN example `1234`. Once encoded, this PIN gives in hexadecimal: `04 8D 3F FF`.
The zone protected by the secret code is **ATD**; on my card, the table seen earlier indicates that the address is `02 F8`:

```text
[david@castlebbs tools]$ scsend "I BC 20 00 00 04 04 8D 3F FF"
I BC 20 00 00 04 09 D5 BF FF
9000
9000: Normal processing

[david@castlebbs tools]$ scsend "I BC 40 00 00 00"
I BC 40 00 00 00
9000
9000: Normal processing

[david@castlebbs tools]$ scsend "O BC B0 02 F8 40"
O BC B0 02 F8 40
-----------------------------------------------------------FF9000
```

#### Reading the 768-bit key

On my card, this key is at address `08 D0`:

```text
[david@castlebbs tools]$ scsend "O BC B0 08 D0 78"
O BC B0 08 D0 78
2E16703A30--------------------------------
```
