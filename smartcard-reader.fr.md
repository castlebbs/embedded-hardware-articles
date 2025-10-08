Here is a **Markdown**-formatted version of that archived article (translated where needed, code blocks preserved):

---

## CP8 smartcard reader: explore the carte bleue (credit card)

_By castlebbs on Tuesday 13 February 2007, 14:57_
_(Original French)_ ([Wayback Machine][1])

> J’ai acheté un lecteur de cartes à puce d’occasion pour 1 euro dans un magasin Toulousain. Ce lecteur assez vieux fonctionne parfaitement sous Linux, j’ai décidé d’écrire cette page pour décrire le travail que j’ai fait autour de ce lecteur. ([Wayback Machine][1])
> L’auteur ne pourra en aucun cas être tenu responsable des préjudices ou dommages de quelque nature que ce soit pouvant résulter de l’utilisation de ses explications ou programmes. ([Wayback Machine][1])

### Connexion du lecteur

#### Alimentation

Le lecteur doit être alimenté en 12 V continu. Je n’ai pas mesuré sa consommation.
J’ai utilisé un transformateur 12 V / 1200 mA (il y a sûrement de la marge :-) ) le connecteur de mon alimentation est un Jack alimentation de 5,5 mm de diamètre, le + au centre. ([Wayback Machine][1])

#### Câble série

Ceci a été mon premier problème car je ne connaissais pas le brochage du lecteur. Le connecteur série du lecteur est une prise RJ45. J’ai démonté le lecteur et identifié le composant **LT1181ACS** qui est le driver RS232 (composant qui génère les bonnes tensions pour le RS232 (+/−12 V)).
À partir du datasheet et un “examen” du routage, j’ai utilisé le brochage suivant : ([Wayback Machine][1])

> La pin 1 est à gauche sur la prise RJ45 femelle, clip vers le bas.
>
> - `5 (rxd)` connectée au `3 (txd)` de la DB9 côté PC
> - `3 (txd)` connectée au `2 (rxd)` de la DB9 côté PC
> - `1 (masse)` connectée au `5 (masse)` de la DB9 côté PC
>
> La pin `T1out` du driver série est connectée à la pin `7` de la RJ45, je ne sais pas à quoi elle sert et ne l’ai pas câblée. ([Wayback Machine][1])

Consulter le **brochage des DB9 et DB25** pour fabriquer le connecteur côté PC. ([Wayback Machine][1])

### Fonctionnement sous Linux

Après recherches sur Internet, ce lecteur dialogue avec l’ordinateur selon le protocole **TLP 224**.
Un driver permettant d’envoyer des commandes ISO à la carte existe et fonctionne : _[http://www.gemplus.com/techno/tlp_drivers/](http://www.gemplus.com/techno/tlp_drivers/)_ ([Wayback Machine][1])

Ce driver n’est hélas pas compatible PC/SC, c’est à dire qu’il n’est pas possible d’utiliser les applications écrites pour le framework **MUSCLE**. ([Wayback Machine][1])

#### Installation

1. Télécharger `libtlp_1.0.5.tar.gz` ([Wayback Machine][1])
2. Dans le répertoire `src` :

   ```bash
   make && make install
   ```

   (modifier le fichier Makefile si vous souhaitez changer le préfix d’installation) ([Wayback Machine][1])

3. Créer un lien `/dev/screader` par exemple pour le port série COM1 :

   ````bash
   ln -s /dev/ttyS0 /dev/screader
   ``` :contentReference[oaicite:11]{index=11}
   ````

#### Test

Insérer une carte dans le lecteur, envoyer un **reset** à la carte :

```text
[david@castlebbs tools]$ scsend R
R
C0651125080038046C9000
```

La carte a été initialisée, la LED orange du lecteur s’allume. Pour éteindre la carte :

```text
[david@castlebbs tools]$ scsend P
P
```

La LED est maintenant éteinte. ([Wayback Machine][1])

### Fonctionnement sous Windows

Je n’ai jamais testé le lecteur sous Windows. On peut trouver sur le forum fr.comp.carte-a-puce des informations dont ce **thread**.
J’ai lu que des personnes utilisent un **programme sous Windows**. ([Wayback Machine][1])

### Mise en pratique : exploration d’une carte bleue

Cette partie explique comment procéder à la lecture de sa carte bleue comme décrit par Laurent PELE, mais avec le lecteur Bull CP8 et sous Linux.
**Utilisation à des fins éducatives exclusivement.** Tout usage détourné donnera lieu à des poursuites selon l’article L 335-3 alinéa 2 du code de la propriété intellectuelle. ([Wayback Machine][1])

#### Lecture zone libre de la puce

Première lecture de la zone libre de la puce :

```text
[david@castlebbs tools]$ scsend "O BC B0 09 E0 20"
O BC B0 09 E0 20
3FE52002084D00250935FF671C2B9F55000000000000000000000000000000009000
9000: Normal processing
```

#### Zones d’adressage

Lecture de la zone d’adressage ADL :

```text
[david@castlebbs tools]$ scsend "O BC B0 09 C8 08"
O BC B0 09 C8 08
1FD10BEB0BEB0B7F9000
9000: Normal processing
```

On remarque que sur ma carte de test, les 16 premiers bits `1FD1` ne sont pas les mêmes que sur la page de Laurent PELE, et ne correspondent pas non plus à `1FF4` comme écrit dans la section _“Exploration sur une carte émise depuis novembre 1999”_.
La carte que je teste a été émise en novembre 2001 — cela signifie que l’adressage a été modifié.
Voici le tableau des zones d’adresses pour ma carte :

| Nom zone | Adresse définition | Octets adresse définition | Résultat zone d’adresse |
| -------- | ------------------ | ------------------------- | ----------------------- |
| ADL      | 09C8               | 1FD1                      | 07F0                    |
| ADT      | 09CC               | 0BEB                      | 02F8                    |
| ADC      | 09D0               | 0BEB                      | 02F8                    |
| ADM      | 09D4               | 0B7F                      | 02C0                    |
| AD2      | 09D8               | 0B10                      | 0B10                    |
| ADS      | 09DC               | 08D9                      | 0230                    |
| AD1      | 09E8               | 084D                      | 0210                    |

La manipulation de bits est assez simple : pour obtenir l’adresse de la zone ADL, il suffit d’effectuer l’opération suivante :

```text
(0x1FD1 & 0xFFE0) >> 2
```

Script ayant permis de générer ce tableau :

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

#### Zone identifiant de la puce

Sur ma carte, la zone identifiant du prestataire 02 est stockée à l’adresse `08 58`. Je peux la lire avec :

```text
[david@castlebbs tools]$ scsend "O BC B0 08 58 40"
O BC B0 08 58 40
------------------------------------------------------ …
9000: Normal processing
```

Je n’ai pas mis en ligne la zone identifiant de ma carte. J’ai écrit un petit programme en Python pour décoder les informations de cette zone :

```python
#!/usr/bin/env python
# decode.py – Décode la zone identifiant CB – 2004 – David ROBERT

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

print "Numero de carte : " + newval[2:18]
print "Code usage : " + newval[21:24]
print "Date d’emission : %s/%s (an/mois)" % (newval[24:26], newval[26:28])
print "Code langue : %s (250 = francais)" % newval[28:31]
print "Date d’expiration : %s/%s (an/mois)" % (newval[31:33], newval[33:35])
print "Code numerique ISO : %s  (250 = francs)" % (newval[35:38])
print "Exposant : %s" % newval[38:39]

nom = newval[42:100]
stringnom = ""
compteur = 0
while compteur < len(nom):
    bytevalue = int("%s%s" % (nom[compteur], nom[compteur+1]), 16)
    stringnom = stringnom + "%c" % bytevalue
    compteur = compteur + 2

print "Nom du porteur : " + stringnom
```

**Utilisation du programme :**

```text
[david@castlebbs tools]$ /home/david/decode.py 2E0238F1300497...A9000
Numero de carte : 497-------------
Code usage : 101
Date d’émission : 01/11 (an/mois)
Code langue : 250 (250 = francais)
Date d’expiration : 03/12 (an/mois)
Code numerique ISO : 250  (250 = francs)
Exposant : 5
Nom du porteur : MR DAVID ROBERT
```

#### Zone d’authentification

Lecture de la zone d’authentification (prestataire 3), à l’adresse ADL `07 F0` sur ma carte :

```text
[david@castlebbs tools]$ scsend "O BC B0 07 F0 34"
O BC B0 07 F0 34
-------------------------------------------------------
9000: Normal processing
```

Les “---” sont évidemment remplacés par la valeur d’authentification. ([Wayback Machine][1])

##### Vérification de la valeur d’authentification de la puce

J’ai dû retirer les 4 octets `9000` à la fin et extriare les quartets de `3` tous les 4 octets.
Le code Python suivant permet de préparer la valeur (`val` contient la valeur d’authentification sous forme de chaîne) :

```python
for quarte in val[:-4]:
    if pointeur % 8 != 0:
        newval = newval + quarte
    pointeur = pointeur + 1
print newval
```

Pour les calculs, un simple interpréteur Python suffit :

```python
produit = 213598703592091008...
authent = 0x000074..
"Resultat : %x" % (authent**3 % produit)
```

#### Lire zone protégée par code confidentiel

Prenons comme exemple le code confidentiel `1234`. Une fois encodé, ce code donne en hexadécimal : `04 8D 3F FF`.
La zone protégée par code secret est **ATD**, sur ma carte, le tableau vu précédemment indique que l’adresse est `02 F8` :

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

#### Lecture de la clef 768 bits

Sur ma carte, cette clef est à l’adresse `08 D0` :

```text
[david@castlebbs tools]$ scsend "O BC B0 08 D0 78"
O BC B0 08 D0 78
2E16703A30--------------------------------
```
