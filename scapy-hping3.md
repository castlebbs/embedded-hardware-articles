# scapy vs hping3: ISN Distribution Spectrograph

_By castlebbs on Wednesday 21 March 2007, 17:58_ – Tools / Permalink

`scapy` and `hping3` are network packet manipulation tools. They are Swiss Army knives for crafting, sending, and receiving packets. As I am particularly interested in Python, I naturally gravitated towards scapy. There are other tools and libraries for Python (for example, _impaquet_, pcapy), which I will present in other posts. Hping3 is another very powerful tool that includes a tcl interpreter; so I was interested in comparing these two products.

This post presents my initial comparison and is not meant to be exhaustive. It is just here to share my first impressions.

- scapy
- hping3

---

To start my comparison, I decided to rewrite the **isn-spectrogram** program (originally written in tcl/hping3) in Python/Scapy. Packet creation, sending, and receiving were rewritten with scapy, and the graphical window was rewritten with Tkinter (since isn-spectrogram uses Tk, the port was easy :-)). This program analyzes the sequence numbers returned by the recipient during the initialization of a TCP connection.

This program draws a spectrogram representing the distribution of sequence number gaps returned between different packets. These gaps correspond to random increments, and the wider the spectrum, the harder it is to predict the sequence numbers.

**Linux 2.4.27 spectrogram:**  
![linux spectrogram](/path/to/linux.png)

**Zyxel router spectrogram (!):**  
![zyxel spectrogram](/path/to/zyxel.png)

---

## Code Sections & Comparison

### hping3: construction, sending, and receiving

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

Two functions are used, one to send packets (`sendsyn`), the other to receive responses (`recvsynack`). These functions are called by the tcl event handler: one packet is sent every millisecond, and response packets are received every 10 milliseconds. The hping function used to send is `send`, and `recv` is used to receive. These two functions handle tcl lists in APD format.

---

### Scapy: first version

```python
def sendSyns():
   "Send SYN packets, receive SYN-ACK"
   port = 1
   while True:
      seq = sr1(IP(dst=hostname,ttl=255)/TCP(flags="S",sport=port,dport=dport),verbose=0).seq
      diff = abs(oldseq - seq)
      oldseq = seq
      port += 1
      displayPoint(diff)
```

The advantage of scapy here is:

```python
seq = sr1(IP(dst=hostname,ttl=255)/TCP(flags="S",sport=port,dport=dport),verbose=0).seq
```

In a single line, you build a packet, send it, receive the response, and extract the sequence number. In this Python program, there is a single function to send and receive packets: `sendSyns()`.

This function is executed in a thread (we do not use the tcl event handler concept here).

The scapy function `sr1` allows you to send and receive a packet — very convenient. But in our program, it is too slow. You need to send several thousand packets to get our spectrograms, and it is much too slow to iterate over a scapy `sr1` function that only sends a new packet after receiving the previous response.

So I used another method described below. But to be honest, in all cases it seems that scapy is **much slower** than hping3 for creating and sending packets. Speed is probably not the most important criterion, but the difference is significant enough to be noted.

---

### Sending 10,000 packets: hping3 vs Scapy

**With hping3:**

```tcl
hping3> for { set i 0 } { $i < 10000 } { incr i } { hping send "ip(saddr=192.168.1.2,daddr=192.168.1.1,ttl=255)+tcp(sport=$i dport=222,flags=s)" }
```

This sends 10,000 TCP packets with the SYN flag and the source port value incremented for each packet. This operation takes less than two seconds on the author's machine.

**With Scapy:**

```python
>>> for i in range(10000):  send(IP(src="192.168.1.2",dst="192.168.1.1",ttl=255)/TCP(sport=i,dport=222,flags="S"),verbose=0,inter=0)
```

or

```python
>>> send(IP(src="192.168.1.2",dst="192.168.1.1",ttl=255)/TCP(sport=range(10000),dport=222,flags="S"),verbose=0,inter=0)
```

The equivalent in scapy will take more than 3 minutes to send the 10,000 packets.

In short, as we can see: generating spectrograms with scapy is slower, but with the `sc1` function, it becomes completely unusable. Here is the adopted solution:

```python
def sendSyns():
    "Send SYN packets, receive SYN-ACK"
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

Instead of using `sr1`, I use the `sr` function to send several packets at once and then receive all the responses. The parameter `sport=range(sport, sport+nbsyn)` provides a list of source ports to the TCP object's `sport` argument, which generates several packets at once. `nbsyn` is the number of packets to send in one go.

You gain speed compared to `sr1` since you send "blocks" of `nbsyn` packets without waiting for the response. The author tested different values (which can be specified on the command line), but the graph creation is still much slower than with the hping3 program.

---

### Comparative Summary

| Criterion     | Winner / Comment          |
| ------------- | ------------------------- |
| Lines of code | scapy — more concise code |
| Speed         | hping3 — much faster      |

---

## Complete Program

```python
#!/usr/bin/env python
# -*- coding: iso-8859-15 -*-
# David ROBERT david@ombrepixel.com
# isn-scaptogram v0.1
# Strongly inspired by isn-spectrogram.htcl (hping3 project)
# To compare the features of hping3 and python/scapy scripts

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
    print " scale : spectrogram scale"
    print " blocks : number of SYN packets in a scapy block"
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
                   "Host : %s, Port : %s SYNs per block: %d" \
                   % (hostname, dport, nbsyn), anchor=W)
canvas.create_line(1, 20, 800, 20, fill="#FFFFFF")

def sendSyns():
    "Send SYN packets, receive SYN-ACK"
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
    "Display the line on the spectrogram"
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
