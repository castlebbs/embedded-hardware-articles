# scapy vs hping3: ISN distribution spectrogram

_By castlebbs on Wednesday, 21 March 2007, 17:58_ – Tools / Permalink

`scapy` and `hping3` are network packet manipulation tools. They are Swiss Army knives for creating, sending, and receiving packets. I am particularly interested in how they can be used to study TCP sequence numbers, especially ISN generation.

This post presents my first steps in comparison and is not intended to be exhaustive on the subject. It is simply there to collect my first impressions.

- scapy
- hping3

---

To begin my comparison, I decided to rewrite the **isn-spectrogram** program, written in tcl/hping3, in Python/Scapy. Packet creation, sending, and reception were rewritten using scapy.

This program draws a spectrogram representing the distribution of differences in sequence numbers returned between the various packets. These differences correspond to increments in the initial sequence numbers (ISN) chosen by the remote host.

**Linux 2.4.27 spectrogram:**  
<img width="490" height="352" alt="linux" src="https://github.com/user-attachments/assets/63631443-f013-4210-90be-2bc1d6b93b11" />

**Spectrogram of a Zyxel router (!):**  
<img width="489" height="352" alt="zyxel" src="https://github.com/user-attachments/assets/974454a0-14cf-4098-88c0-a7f265382782" />

---

## Code snippets & comparison

### hping3: construction, sending, and reception

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

Two functions are used: one to send packets (`sendsyn`), and another to receive replies (`recvsynack`). These functions are called by Tcl’s event handler.

---

### Scapy: first version

```python
def sendSyns():
   "Sends SYN packets and receives SYN-ACKs"
   port = 1
   while True:
      seq = sr1(IP(dst=hostname,ttl=255)/TCP(flags="S",sport=port,dport=dport),verbose=0).seq
      diff = abs(oldseq - seq)
      oldseq = seq
      port += 1
      displayPoint(diff)
```

The advantage of scapy is here:

```python
seq = sr1(IP(dst=hostname,ttl=255)/TCP(flags="S",sport=port,dport=dport),verbose=0).seq
```

In a single line, we build a packet, send it, receive the response, and extract the sequence number. In this Python program, there will be a single function for sending and receiving packets.

This function runs in a thread (we are not using Tcl’s event handler here).

The `sr1` scapy function allows us to send and receive a packet — very convenient. But in the case of our program, it is too slow. We need to send several thousand packets to obtain the distribution we’re interested in, so the delay becomes significant.

I therefore used another method described below. But to be honest, in all cases it seems that scapy is much slower than hping3 for packet creation and sending.

---

### Sending 10,000 packets: hping3 vs Scapy

**With hping3:**

```tcl
hping3> for { set i 0 } { $i < 10000 } { incr i } { hping send "ip(saddr=192.168.1.2,daddr=192.168.1.1,ttl=255)+tcp(sport=$i dport=222,flags=s)" }
```

This sends 10,000 TCP packets with the SYN flag set, with the source port incremented on each packet. This takes less than two seconds on the author’s machine.

**With Scapy:**

```python
>>> for i in range(10000):  send(IP(src="192.168.1.2",dst="192.168.1.1",ttl=255)/TCP(sport=i,dport=222,flags="S"),verbose=0,inter=0)
```

or

```python
>>> send(IP(src="192.168.1.2",dst="192.168.1.1",ttl=255)/TCP(sport=range(10000),dport=222,flags="S"),verbose=0,inter=0)
```

The equivalent under scapy takes more than 3 minutes to send the 10,000 packets.

So we’ve understood the point: generating spectrograms with scapy is slower, but with the `sr1` function it becomes completely unusable. Here is the solution that was adopted:

```python
def sendSyns():
    "Sends SYN packets and receives SYN-ACKs"
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

Instead of using `sr1`, I use the `sr` function to send several packets at once and receive the set of responses afterward. The `sport=range(sport, sport+nbsyn)` parameter makes it possible to send `nbsyn` packets in a block.

This is faster than `sr1` because packets are sent in blocks of `nbsyn` without waiting for each response individually. The author tested different values (configurable at runtime) to find the best compromise between speed and reliability.

---

### Comparative summary

| Criterion | Winner / Comment |
| --------- | ---------------- |
| Number of lines of code | scapy — more concise code |
| Speed | hping3 — significantly faster |

---

## Complete program

```python
#!/usr/bin/env python
# -*- coding: iso-8859-15 -*-
# David ROBERT david@ombrepixel.com
# isn-scaptogram v0.1
# Heavily inspired by isn-spectrogram.htcl (hping3 project)
# To compare the features of the hping3 and python/scapy scripts

from scapy import sr, IP, TCP
from Tkinter import *
import threading, sys, time

try:
    hostname = sys.argv[1]
    dport = int(sys.argv[2])
    div = int(sys.argv[3])
    nbsyn = int(sys.argv[4])
except:
    print "Usage: spectoscapy.py <host> <open-tcp-port> <scale> <blocks>"
    print "Example: spectoscapy.py www.example.com 80 100000 10"
    print " scale: scale of the spectrogram"
    print " blocks: number of SYN packets in a scapy block"
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
                   "Host : %s, Port : %s Number of SYN per block: %d" \
                   % (hostname, dport, nbsyn), anchor=W)
canvas.create_line(1, 20, 800, 20, fill="#FFFFFF")

def sendSyns():
    "Sends SYN packets and receives SYN-ACKs"
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
    "Draws the line on the spectrogram"
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
