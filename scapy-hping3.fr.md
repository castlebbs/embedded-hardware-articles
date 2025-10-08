# scapy vs hping3 : spectrographe de distribution ISN

_By castlebbs on Wednesday 21 March 2007, 17:58_ – Tools / Permalink

`scapy` et `hping3` sont des outils de manipulation de paquets réseau. Ce sont des couteaux suisses de la fabrication, de l’envoi et de la réception de paquets. M’intéressant particulièrement à Python, je me suis naturellement intéressé à scapy. Il existe d’autres outils et bibliothèques pour python (par exemple _impaquet_, pcapy). Je les présenterai dans d’autres billets. Hping3 est un autre outil très puissant intégrant un interpréteur tcl ; je me suis donc intéressé à comparer ces deux produits.

Ce billet présente mes débuts de comparaison et n’est pas fait pour être exhaustif sur le sujet. Il est juste là pour recueillir mes premières impressions.

- scapy
- hping3

---

Pour commencer ma comparaison j’ai décidé d’écrire en Python/Scapy le programme **isn-spectrogram** écrit en tcl/hping3. La création de paquet, envoi et réception a été réécrite avec scapy, la fenêtre graphique réécrite avec Tkinter (isn-spectrogram utilisant Tk, le portage a été facile :-)). Ce programme analyse les numéros de séquence renvoyés par le destinataire lors de l’initialisation d’une connexion TCP.

Ce programme dessine un spectrogramme qui représente la distribution des écarts de numéros de séquence renvoyés entre les différents paquets. Ces écarts correspondent à des incréments aléatoires, et plus le spectre sera large, plus il sera difficile de déterminer les numéros de séquences.

**Spectrogramme linux 2.4.27 :**  
<img width="490" height="352" alt="linux" src="https://github.com/user-attachments/assets/63631443-f013-4210-90be-2bc1d6b93b11" />

**Spectrogramme d’un routeur Zyxel (!) :**  
<img width="489" height="352" alt="zyxel" src="https://github.com/user-attachments/assets/974454a0-14cf-4098-88c0-a7f265382782" />


---

## Parties de code & comparaison

### hping3 : construction, émission et réception

```tcl
proc sendsyn {} {
    global sport dport myip target
    append syn "ip(saddr=$myip,daddr=$target,ttl=255)+"
    append syn "tcp(sport=$sport,dport=$dport,flags=s)"
    hping send $syn
    incr sport
    after 1 sendsyn
}
proc recvsynack {} {
    global lastisn relative_attractor
    set packets [hping recv eth0 0 0]
    foreach p $packets {
        if {![hping hasfield tcp flags $p]} continue
        set isn [hping getfield tcp seq $p]
        if {$relative_attractor} {
            set tisn [expr abs($isn-$lastisn)]
            set lastisn $isn
            set isn $tisn
        }
        # puts "ISN: $isn"
        displaypoint $isn
    }
    after 10 recvsynack
}
```

Deux fonctions sont utilisées, une pour envoyer des paquets (`sendsyn`), l’autre pour recevoir les réponses (`recvsynack`). Ces fonctions sont appelées par le gestionnaire d’événements de tcl : envoi d’un paquet chaque milliseconde, réception des paquets de réponse toutes les 10 millisecondes. La fonction hping utilisée pour envoyer est `send`, et `recv` pour recevoir. Ces deux fonctions traitent des listes tcl au format APD.

---

### Scapy : première version

```python
def sendSyns():
   "Envoie des paquets SYN, reception de SYN-ACK"
   port = 1
   while True:
      seq = sr1(IP(dst=hostname,ttl=255)/TCP(flags="S",sport=port,dport=dport),verbose=0).seq
      diff = abs(oldseq - seq)
      oldseq = seq
      port += 1
      displayPoint(diff)
```

L’avantage de scapy est ici :

```python
seq = sr1(IP(dst=hostname,ttl=255)/TCP(flags="S",sport=port,dport=dport),verbose=0).seq
```

En une seule ligne, on construit un paquet, on l’envoie, on récupère la réponse et on en extrait le numéro de séquence. Dans ce programme Python, il y aura une unique fonction pour envoyer et recevoir les paquets : `sendSyns()`.

Cette fonction est exécutée dans un thread (on n’utilise pas ici la notion de gestionnaire d’événements tcl).

La fonction scapy `sr1` permet d’envoyer et de recevoir un paquet — très pratique. Mais dans le cas de notre programme, c’est trop lent. Il faut envoyer plusieurs milliers de paquets pour obtenir nos spectrogrammes, et c’est beaucoup trop lent d’itérer sur une fonction `sr1` de scapy qui n’envoie un nouveau paquet qu’après avoir reçu la réponse du paquet précédent.

J’ai donc utilisé une autre méthode décrite ci-dessous. Mais pour être honnête, dans tous les cas il semble que scapy soit **beaucoup plus lent** que hping3 pour ce qui concerne la création et l’envoi de paquets. La vitesse n’est sûrement pas le critère le plus important, mais la différence est suffisamment marquée pour être notée.

---

### Envoi de 10 000 paquets : hping3 vs Scapy

**Sous hping3 :**

```tcl
hping3> for { set i 0 } { $i < 10000 } { incr i } { hping send "ip(saddr=192.168.1.2,daddr=192.168.1.1,ttl=255)+tcp(sport=$i dport=222,flags=s)" }
```

Cela envoie 10 000 paquets TCP avec le flag SYN et la valeur du port source incrémentée à chaque paquet. Cette opération prend moins de deux secondes sur le poste de l’auteur.

**Sous Scapy :**

```python
>>> for i in range(10000):  send(IP(src="192.168.1.2",dst="192.168.1.1",ttl=255)/TCP(sport=i,dport=222,flags="S"),verbose=0,inter=0)
```

ou

```python
>>> send(IP(src="192.168.1.2",dst="192.168.1.1",ttl=255)/TCP(sport=range(10000),dport=222,flags="S"),verbose=0,inter=0)
```

L’équivalent sous scapy mettra plus de 3 minutes pour envoyer les 10 000 paquets.

Bref, on l’a compris : la génération de spectrogrammes avec scapy est plus lente, mais avec la fonction `sc1,` cela devient carrément inutilisable. Voici la solution adoptée :

```python
def sendSyns():
    "Envoie des paquets SYN, reception de SYN-ACK"
    sport = 1
    while running == 1:
        r = sr(IP(dst=hostname,ttl=255)/
               TCP(flags="S", sport=range(sport, sport+nbsyn), dport=dport),
               verbose=0, timeout=0)
        sport += nbsyn
        for snd, rcv in r[0]:
            diff = abs(oldseq - rcv.seq)
            oldseq = rcv.seq
            displayPoint(diff)
```

Au lieu d’utiliser `sr1`, j’utilise la fonction `sr` pour envoyer plusieurs paquets en une fois et recevoir l’ensemble des réponses ensuite. Le paramètre `sport=range(sport, sport+nbsyn)` fournit une liste de ports source à l’argument `sport` de l’objet TCP, ce qui génère plusieurs paquets à la fois. `nbsyn` correspond au nombre de paquets à envoyer d’un seul coup.

On gagne en vitesse par rapport à `sr1` dans la mesure où on envoie par « blocs » de `nbsyn` paquets sans attendre la réponse. L’auteur a testé différentes valeurs (spécifiables en ligne de commande), mais la création du graphe reste quand même bien plus lente qu’avec le programme sous hping3.

---

### Résumé comparatif

| Critère                  | Gagnant / Commentaire          |
| ------------------------ | ------------------------------ |
| Nombre de lignes de code | scapy — code plus concis       |
| Vitesse                  | hping3 — nettement plus rapide |

---

## Programme complet

```python
#!/usr/bin/env python
# -*- coding: iso-8859-15 -*-
# David ROBERT david@ombrepixel.com
# isn-scaptogram v0.1
# Fortement inspiré de isn-spectrogram.htcl (projet hping3)
# Pour comparer les fonctionnalités des scripts hping3 et python/scapy

from scapy import sr, IP, TCP
from Tkinter import *
import threading, sys, time

try:
    hostname = sys.argv[1]
    dport = int(sys.argv[2])
    div = int(sys.argv[3])
    nbsyn = int(sys.argv[4])
except:
    print "Utilisation: spectoscapy.py <host> <open-tcp-port> <scale> <blocks>"
    print "Exemple: spectoscapy.py www.example.com 80 100000 10"
    print " scale : échelle du spectrogramme"
    print " blocks : nombre de paquets SYN dans un bloc scapy"
    sys.exit(1)

def quit():
    global running
    running = 0

running = 1
pastcol = {}
root = Tk()
root.title("isn-scaptogram")
root.config(background="#000000")
frame = Frame(root)
frame.config(background="#000000")
frame.pack(side=TOP)
button = Button(frame, text="QUIT", fg="red", command=quit)
button.pack(side=LEFT)
canvas = Canvas(root)
canvas.config(width=800, height=300)
canvas.config(background="#000000")
canvas.pack(fill=BOTH, expand=TRUE)
canvas.create_rectangle(40, 250, 139, 250, fill="#FFFFFF", width=0)
canvas.create_text(90, 270, fill="#FFFFFF", text=div*100)
canvas.create_text(10, 10, fill="#FFFFFF", text= \
                   "Host : %s, Port : %s Nombre de SYN par blocs: %d" \
                   % (hostname, dport, nbsyn), anchor=W)
canvas.create_line(1, 20, 800, 20, fill="#FFFFFF")

def sendSyns():
    "Envoie des paquets SYN, réception de SYN-ACK"
    oldseq = 0
    sport = 1
    while running == 1:
        r = sr(IP(dst=hostname, ttl=255) /
               TCP(flags="S", sport=range(sport, sport+nbsyn), dport=dport),
               verbose=0, timeout=0)
        sport += nbsyn
        for snd, rcv in r[0]:
            diff = abs(oldseq - rcv.seq)
            oldseq = rcv.seq
            displayPoint(diff)
    frame.quit()

def displayPoint(isn):
    "Affiche la ligne sur le spectrogramme"
    isn = isn / div
    y = 50
    x = isn
    if not pastcol.has_key((x, y)):
        pastcol[(x, y)] = 0
        graylevel = 0
    else:
        pastcol[(x, y)] += 10
        graylevel = pastcol[(x, y)]
    if graylevel >= 256*3:
        graylevel = 256*3 - 1
    if graylevel <= 255:
        b = graylevel
        g = r = 0
    elif graylevel <= 511:
        b = 0
        g = graylevel - 256
        r = 255
    elif graylevel <= 767:
        b = g = 255
        r = graylevel - 512
    canvas.create_rectangle(x, y, x+1, y+170,
                            fill="#%02X%02X%02X" % (r, g, b),
                            width=0)

t = threading.Thread(target=sendSyns, args=())
t.start()

root.mainloop()
```
