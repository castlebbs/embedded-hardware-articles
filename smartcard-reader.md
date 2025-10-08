# CP8 smartcard reader: exploring the Carte Bleue (credit card)

_By castlebbs on Tuesday, 13 February 2007, 14:57._
(Archived original: Wayback Machine). ([Wayback Machine][1])

> I bought a used smartcard reader for 1 euro in a Toulouse shop. This rather old reader works perfectly under Linux; I decided to write this page to describe the work I did around this reader.
>
> The author cannot be held responsible for any prejudice or damage of any kind resulting from the use of these explanations or programs.

---

## Connecting the reader

### Power

The reader must be powered with 12 V DC. I did not measure its consumption. I used a 12 V / 1200 mA transformer (there is probably some margin :-) ). The power connector on my supply is a 5.5 mm jack, positive on the center pin. ([Wayback Machine][1])

### Serial cable

This was my first problem because I did not know the reader’s pinout. The reader’s serial connector is an RJ45 jack. I opened the reader and identified the chip **LT1181ACS**, which is the RS-232 driver (component that generates the proper ±12 V RS-232 levels). From the datasheet and a “visual” inspection of the PCB routing, I used the following pinout: ([Wayback Machine][1])

> Pin 1 is on the left of the RJ45 female connector with the clip facing down.
>
> - `5 (rxd)` connected to `3 (txd)` of the PC DB9
> - `3 (txd)` connected to `2 (rxd)` of the PC DB9
> - `1 (ground)` connected to `5 (ground)` of the PC DB9
>
> The `T1out` pin of the serial driver is connected to pin `7` of the RJ45; I don’t know what it’s used for and I did not wire it.

Consult DB9 / DB25 pinouts when building the PC-side connector. ([Wayback Machine][1])

---

## Operation under Linux

After research on the Internet, this reader speaks the **TLP 224** protocol to the computer. A driver allowing ISO commands to be sent to the card exists and works: `libtlp` (e.g. `libtlp_1.0.5`). Note that this driver is not PC/SC compatible, which means applications written for the MUSCLE framework cannot be used directly. ([Wayback Machine][1])

### Installation (summary from the article)

1. Download `libtlp_1.0.5.tar.gz`.
2. In the `src` directory:

   ```bash
   make && make install
   ```

   (edit the `Makefile` if you want to change the installation prefix).

3. Create a symlink such as `/dev/screader` pointing to the serial device, e.g.:

   ````bash
   ln -s /dev/ttyS0 /dev/screader
   ``` :contentReference[oaicite:5]{index=5}
   ````

### Test

Insert a card into the reader and send a reset to the card:

```text
[david@castlebbs tools]$ scsend R
R
C0651125080038046C9000
```

The card initialized and the reader’s orange LED lights. To power off the card:

```text
[david@castlebbs tools]$ scsend P
P
```

The LED is now off. ([Wayback Machine][1])

---

## Operation under Windows

The author never tested the reader under Windows. He references a thread on the French `fr.comp.carte-a-puce` forum and mentions that some people use a Windows program. ([Wayback Machine][1])

---

## Practical example: exploring a Carte Bleue

This section explains how the author read his Carte Bleue (as described by Laurent PELE) but using the Bull CP8 reader under Linux. The author stresses **educational use only**; misuse may lead to prosecution under intellectual property / other laws. ([Wayback Machine][1])

> _Note: the content that follows reproduces commands, scripts and examples from the archived post. It describes reading chip data and simple decoding scripts. The original author framed this as educational — please use responsibly and lawfully._

### Reading the chip’s free zone

First read of the chip free zone:

```text
[david@castlebbs tools]$ scsend "O BC B0 09 E0 20"
O BC B0 09 E0 20
3FE52002084D00250935FF671C2B9F55000000000000000000000000000000009000
9000: Normal processing
```

### Addressing zones

Read the ADL addressing zone:

```text
[david@castlebbs tools]$ scsend "O BC B0 09 C8 08"
O BC B0 09 C8 08
1FD10BEB0BEB0B7F9000
9000: Normal processing
```

On the author’s test card, the first 16 bits `1FD1` are different from those on Laurent PELE’s page and also differ from `1FF4` referenced elsewhere; the card used for testing was issued November 2001, meaning addressing changed. The author shows this table of address zones for his card: ([Wayback Machine][1])

| Zone name | Definition address | Address definition bytes | Resulting address |
| --------- | -----------------: | -----------------------: | ----------------: |
| ADL       |             `09C8` |                   `1FD1` |            `07F0` |
| ADT       |             `09CC` |                   `0BEB` |            `02F8` |
| ADC       |             `09D0` |                   `0BEB` |            `02F8` |
| ADM       |             `09D4` |                   `0B7F` |            `02C0` |
| AD2       |             `09D8` |                   `0B10` |            `0B10` |
| ADS       |             `09DC` |                   `08D9` |            `0230` |
| AD1       |             `09E8` |                   `084D` |            `0210` |

Bit manipulation is straightforward; to obtain the ADL zone address:

```text
(0x1FD1 & 0xFFE0) >> 2
```

Script used to generate the table (shell + `gawk`):

```sh
#!/bin/sh

# Calcul des zones d’adresses. 2004 - David ROBERT
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

(Original comments and code are preserved.) ([Wayback Machine][1])

### Chip identifier zone

On the author’s card, the provider identifier zone (provider 02) is stored at address `08 58`. He reads it like this:

```text
[david@castlebbs tools]$ scsend "O BC B0 08 58 40"
O BC B0 08 58 40
------------------------------------------------------ …
9000: Normal processing
```

The author did not publish his card’s identifier zone. He wrote a small Python script to decode that zone: ([Wayback Machine][1])

```python
#!/usr/bin/env python
# decode.py – Decode CB identifier zone – 2004 – David ROBERT

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
print "Expiry date : %s/%s (year/month)" % (newval[31:33], newval[33:35])
print "Numeric ISO code : %s  (250 = francs)" % (newval[35:38])
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

Example usage and output shown by the author:

```text
[david@castlebbs tools]$ /home/david/decode.py 2E0238F1300497...A9000
Card number : 497-------------
Usage code : 101
Issue date : 01/11 (year/month)
Language code : 250 (250 = French)
Expiry date : 03/12 (year/month)
Numeric ISO code : 250  (250 = francs)
Exponent : 5
Cardholder name : MR DAVID ROBERT
```

> _The author’s example shows partially redacted card number data._ ([Wayback Machine][1])

### Authentication zone

Reading the authentication zone (provider 3) — on his card it’s at ADL `07 F0`:

```text
[david@castlebbs tools]$ scsend "O BC B0 07 F0 34"
O BC B0 07 F0 34
-------------------------------------------------------
9000: Normal processing
```

The `---` in the output indicate the authentication value returned by the card. ([Wayback Machine][1])

#### Verifying the chip authentication value

The author removed the trailing `9000` bytes and extracted every 4th quartet; he shows a small Python snippet used to prepare the value (variable `val` contains the authentication value string):

```python
for quarte in val[:-4]:
    if pointeur % 8 != 0:
        newval = newval + quarte
    pointeur = pointeur + 1
print newval
```

For calculations the author used a Python interpreter:

```python
produit = 213598703592091008...
authent = 0x000074..
"Result : %x" % (authent**3 % produit)
```

### Read zone protected by PIN

Example: PIN `1234` encoded in hex becomes `04 8D 3F FF`. The secret-protected zone is ATD; on his card the table above indicates the address `02 F8`. Example interaction:

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

### Reading the 768-bit key

On the author’s card this key is at address `08 D0`:

```text
[david@castlebbs tools]$ scsend "O BC B0 08 D0 78"
O BC B0 08 D0 78
2E16703A30--------------------------------
```
