---
title: "Scoreboard Reverse Engineering"
description: ""
date: 2025-02-14T20:01:13-05:00
draft: true
tags: ["hacking", "reverse engineering"]
toc: true
---


## Premise & Goal
I am attempting to hack the scoreboard at Heatwave Area, or well kind of. The goal for now is just to be able to read the current state of the scoreboard and be able to utilize that information for other projects. [I’ve been streaming some of my roller games here and I have a score bug on my stream](https://www.twitch.tv/turkeydev/clip/FilthyTawdryDonutKappaClaus-CPZQF9Gd1WU4-RtU). This score bug is actually driven from the site that is used to track the  league, but it would be much simpler and be better to eliminate the middleman and drive the score bug purely off the scoreboard that’s already here. Additionally there are some cool things we’d love to explore here at the rink such as adding LED’s that can do effects and be driven off the current state of the scoreboard.

Now Varsity Scoreboards does have a partnership with another company to give you 3rd party access to some of this data, but it requires a $500 piece of equipment and a $400 a year subscription. Suffice to say that is not in the budget and needing a 3rd party to be able to get the data out of the scoreboard the rink already owns is just….. Annoying, but this does show it is possible.

So come along as I attempt to reverse engineer this scoreboard and if you know anything feel free to leave a comment. I am figuring this all out as I go and would love some helpful insights from those smarter than me at this.

Quick disclaimer, I know I could use some sort of AI image to text algorithm on the scoreboard, but AI takes a lot more compute power than reverse engineering the communications and I’m just not someone who cares to mess with AI based projects. It’s a totally valid solution, just not the one I’m exploring.

## Wireless Tangent
Initially when this scoreboard was installed the controller, which handles all the user input and drivers what the scoreboard displays, was wireless and utilized a 2.4 giga hertz LoRa Transceiver module for communications.

![Motherboard](./motherboard.png)

I purchased the same module and started to try and use it to intercept the wireless packets. I didn’t get very far here, and I suspect the packets are encrypted, but the wireless connection between the scoreboard and controller was very flaky to say the least. There were constant disconnections between the scoreboard and controller that drove us up the wall. 

## Wired Communication Standard
Ultimately we settled to wire the controller to the scoreboard, so I moved my efforts to trying to reverse engineer the wired connection. This is where the project really starts. 

First the connection utilizes 5 pin din connectors which realistically tells me nothing as there isn’t any sort of standards around the pinouts or what they are even carry. Even if there are, there’s always a chance the company could be using the cables outside of spec too. Opening the controller gives me much more information as I can clearly see traces running from 2 of the pins on these din connectors to an IC. 

![Image of the pin traces](./pin_traces.png)

Sadly the IC’s have some sort of paint or covering over the IC’s markings, but with some careful lighting tricks and guesses I can make out the text VP12, which after googling leads me to [this family of RS-485 transceivers](https://www.ti.com/lit/ds/symlink/sn65hvd10.pdf).

## Intercepting Packets
Now that I know what I am looking for with respect to the communications over the wired signal, I need to determine the pins carrying the communications and the protocol the controller and scoreboard are using to communicate. RS-485 is a communication that uses 2 wires in a half-duplex communication mode. Determining what 2 pins carry the data signal is mainly just following what pins the pcb traces go to on connector and then plugging into those two pins. For now I'll ignore the other 3 pins.

Next I can utilize a cheap logic analyzer and [Sigrok](https://sigrok.org/wiki/Main_Page) to determine the baud rate and protocol so that I can start loading the packets in code. I had never used a logic analyzer or this software before, but overall it wasn't too hard to use. Side note, you can actually use a Pi Pico as a logic analyzer if you so want and I did try it out for fun. Ultimately the cheap logic analyzer I got can handle way higher transmission speeds and baud rates for essentially the same price, but the Pi Pico would have worked for this scenario in the end.

With the logic analyzer connected I set Sigrok to scan at 9600 baud rate for TODO samples or equivalent to about TODO seconds for the given baud rate. I ran this and scrubbed through the output and lo and behold data! Now I did have a feeling this data would be using the UART protocol due to it being a very common protocol for both the rs-485 communications and electronics in general so I had Sigrok try and decode these data packets as UART. Unfortunately these data packets did not map cleanly to UART, but I decided to bump the baud rate to 11520 as it is another very common baud rate and shockingly, Sigrok was now showing valid packets for this data! 

## Reading In Full Packets
With the protocol, baud rate, and some other information determined it was now time to move on to reading in a packet with code! For this I decided to use Python and it's serial capabilities to just read in hex values over this serial connection. I wrote up a very basic program to verify I was getting valid hex that matched what sigrok showed on it's decoded packets.

```Python
import serial  # import the module
con = serial.Serial('/dev/ttyUSB0')  # open COM3
con.baudrate = 115200
con.bytesize = 8  # Number of data bits = 8
con.parity = 'N'  # No parity
con.stopbits = 1  # Number of Stop bits = 1
con.timeout = 0.001
while True:
    bytes = con.read(1)
    hex = bytes.hex()
    if bytes:
        print(bytes.hex())
con.close()  # Close the Com port
```

No parsing, no grouping, nothing, just raw hex values to the terminal. After running it I am greeted with a lot of hex characters, but after a short while watching them roll in I start to see a very common 2 char byte and after cross referencing with Sigrok I determined every packet is started with `f6` as a sort of header byte. With that knowledge in hand I improved my program to build strings and output the current string buffer each time an `f6` is received signaling a new string or packet is beginning. Running this I start building up a big list of them and notice that I'm often getting a lot of the same packets over and over. [I built up a very crude parser and started to eliminate packets being logged until I was seeing nothing new in the terminal](https://github.com/TheTurkeyDev/Varsity-Scoreboard-Packet-Parser/blob/05152c53ab156f49164133d7c4f612f0fdb100e8/main.py). From there I started to build up large examples of each of these packets to compare them all together. 

{{< details summary="Packets">}}
```
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3b9a0300006e686
f6e6e65
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f60cdcff0203000828f1bc6b76
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3b9b0300007068706b7062
f60bdcff0201088011333bba
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3b9c0300006d686d5
f6d76
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f60bdcff02010880113404f0
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3b9d030000706872707059
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f60cdcff0203000828f1bc6b76
f60bdcff02010880113409eb
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3b9e0300006b686b6b6b6e
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f60bdcff0201088011340ee6
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3b9f0300006
f686f576f75
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f60cdcff0203000828f1bc6b76
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3ba00300005868585258be
f60bdcff02010880113413e1
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f607dc01000800be
f614dc0000ff0848f21b3ba10300006e686e506e7d
f60bdcff02010880113418dc
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3ba20300005e68725e5e8a
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f60cdcff0203000828f1bc6b76
f60bdcff0201088011341dd7
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f618ffff0004a304c85fdc170148f21b3b486
f636b657931aa
f607dc01000800be
f614dc0000ff0848f21b3ba30300005a686a5a5a9d
```
{{< /details >}}

At first I was going with the assumption that the second byte was some sort of packet Id as many packets with the same packet ID had very similar packet contents. This held for most of them, but a few of the packets were only 2 bytes long and some had very large variations in the packet length which seemed odd. At one point I ordered all of the packets by what I had been calling their packet Id's and started to notice the packets were getting longer as the packet Id got bigger. 

{{< details summary="Packets Sorted">}}
```
f607dc01000800be
f607dc01000800be
f607dc01000800be
f607dc01000800be
f607dc01000800be
f607dc01000800be
f607dc01000800be
f607dc01000800be
f607dc01000800be
f607dc01000800be
f60bdcff0201088011333bba
f60bdcff02010880113404f0
f60bdcff02010880113409eb
f60bdcff0201088011340ee6
f60bdcff02010880113413e1
f60bdcff02010880113418dc
f60bdcff0201088011341dd7
f60cdcff0203000828f1bc6b76
f60cdcff0203000828f1bc6b76
f60cdcff0203000828f1bc6b76
f60cdcff0203000828f1bc6b76
f614dc0000ff0848f21b3b9a0300006e686
f614dc0000ff0848f21b3b9b0300007068706b7062
f614dc0000ff0848f21b3b9c0300006d686d5
f614dc0000ff0848f21b3b9d030000706872707059
f614dc0000ff0848f21b3b9e0300006b686b6b6b6e
f614dc0000ff0848f21b3b9f0300006
f614dc0000ff0848f21b3ba00300005868585258be
f614dc0000ff0848f21b3ba10300006e686e506e7d
f614dc0000ff0848f21b3ba20300005e68725e5e8a
f614dc0000ff0848f21b3ba30300005a686a5a5a9d
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f618ffff0004a304c85fdc170148f21b3b486
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f636b657931aa
f686f576f75
f6d76
f6e6e65
```
{{< /details >}}

Additionally the Id's where the length of each packet varied slightly between them seemed to have a corresponding small 2, 3, or 4 bytes tiny packet following it (In the list ordered by receive time) that when appended to it made all the packets for that Id the same length. Obviously splitting on the `f6` byte alone is very imperfect and it any packet containing `f6` in its data will cause it to be chopped in half. Furthermore, now that all the packets were sorted, their length often grew bigger by 1 byte, when the Id changed by 1.

It's the packet length!!! I couldn't believe that I hadn't realized this sooner. This makes parsing in packets infinitely easier! [I was now able to parse in full packets to start analyzing them](https://github.com/TheTurkeyDev/Varsity-Scoreboard-Packet-Parser/blob/572c86570c2aa5089511aad664bea9ede78487c2/main.py).

## Decoding The Packets
Now comes the difficult, fun, and tedious aspect of this project. Decoding the packets and determining what they mean. I started with looking at what packets get sent when I spam press a button, only to find out none really do, but I did catch this packet `f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b` that was periodically sending did have slight variations over time. I now decided to specifically target this packet and filtered out the rest. I started with the scoreboard set to all 0's in the 1st period with 14 minutes on the clock. I then pressed most every button on the controller and after each button pressed I logged both the new packet being sent and the new state of the scoreboard. After some time this is what my list looked like

```
f622dcff0200103200000000000000000001000000000000000000ffffffff0000006c - 14 min, per 1
f622dcff0200103300000001000000000001000000000000000000ffffffff0000006a - 14 min, per 1, home 1
f622dcff0200103400000002000000000001000000000000000000ffffffff00000068 - 14 min, per 1, home 2
f622dcff0200103500000002010000000001000000000000000000ffffffff00000066 - 14 min, per 1, home 2, away 1
f622dcff0200103600000002010100000001000000000000000000ffffffff00000064 - 14 min, per 1, home 2, away 1, home shot 1
f622dcff0200103800000002010201000001000000000000000000ffffffff00000060 - 14 min, per 1, home 2, away 1, home shot 2, away shot 1
f622dcff0200103900000002010201000001010100000000000000ffffffff0000005d - 13:54 , per 1, home 2, away 1, home shot 2, away shot 1
f622dcff0200103a00000002010201000002010100000000000000ffffffff0000005b - 13:54 , per 2, home 2, away 1, home shot 2, away shot 1
f622dcff0200103c00000002010201000002010100000000000000f4ffffff01000063 - 13:54 , per 2, home 2, away 1, home shot 2, away shot 1, home pen 1 indicator
f622dcff0200103f00000002010201000002010100000000000000f4f9ffff00000067 - 13:49 , per 2, home 2, away 1, home shot 2, away shot 1, home pen 0 indicator
f622dcff0200104100000002010201000002010100000000000000f4f923ff00010040 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1, away pen 1 indicator
f622dcff0200104300000002010201000002010100000000000000f4f92349000200f3 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1, away pen 2 indicator
f622dcff0200104c00000002010201010002010100000000000000ffffffff00000048 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1, home goal blinking
f622dcff0200104d00000002010201000002010100000000000000ffffffff00000048 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1
f622dcff0200104e00000002010201000102010100000000000000ffffffff00000046 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1, away goal blinking
```

Analyzing the packets it becomes pretty clear that certain bytes correspond to certain bits of data and with this I was able to build up this byte mask

```
    zzvvttuu......ppqq....aabbccddeeffgg..................hhiijjkkllmm..yy
    zz = f6 initiator
    vv = Packet Length
    ttuu = Header. It seems Data can exist between the tt and uu? tt also seems to be able to change?
    pp = Packet counter
    qq = PAcket counter 2?
    aa = Home score
    bb = Away score
    cc = Home Shots
    dd = Away shots
    ee = Home Goal
    ff = Away goal
    gg = Period
    hh = Home 1st penalty number
    ii = Home 2nd penalty number
    jj = Away 1st penalty number
    kk = Away 2nd penalty number
    ll = Home Penalty Indicator
    mm = Away Penalty Indicator
    yy = checksum?
```

After looking at this packet for awhile I moved on to trying to deduce other packets and I did find some more information, but there are 2 vital pieces of information I have not been able to find yet. First I cannot figure out what byte or bytes correspond to some sort of packet Id. This is the last major piece of info missing before I'm really able to reliably parse, and store this information. Secondly I cannot for the world of me find the time for the scoreboard. I have a decent idea where it is, but I cannot figure out how to parse it.

Let's dive into the other packets and what I've been able to decipher so far and then talk about specific data points.


{{< details summary="All the Packets" >}}
```
Unknown Packet (Repeats common):
    f607dc01000800be
    f60740010008005a

Penalty less? Info
    f608dcff020107813c - Home 1st pen empty
    f608dcff020107823b - Home 2nd pen empty
    f608dcff020107833a - Away 1st pen empty
    f608dcff0201078439 - Away 2nd pen empty
    zzvv..........xayy
    zz = f6 initiator
    vv = Packet Length
    yy = checksum?
    a  = Scoreboard Pen slot

Something with Time of Day
    f60bdcff0201088011333bba
    f60bdcff02010880113404f0
    f60bdcff02010880113409eb
    f60bdcff0201088011340ee6
    f60bdcff02010880113413e1
    f60bdcff02010880113418dc
    f60bdcff0201088011341dd7
    f60b22ff020108800e313b79 - 2:49
    f60b22ff020108800e322291 - 2:50 30sec ish
    f60b22ff020108800e32367d - 2:50 50sec ish
    f60b22ff020108800e3304ae - 2:51
    f60b22ff020108800e351d93 - 2:53 30sec ish
    zzvvttuu........hhmmssyy
    zz = f6 initiator
    vv = Packet Length
    yy = checksum?
    ttuu = Header. It seems Data can exist between the tt and uu? tt also seems to be able to change?
    hh = Hours
    mm  = Minutes

Time? (Repeats Common)
    f60cdcff0203000828f1bc6b76 
    f60c40ff02010140d10c001628
    f60c40ff0203000820e5ba6b28
    f60c22ff020101781b0d0016c3 - 14:20
    f60c22ff02010148230d0016eb - 14:21
    f60c22ff020101002f0d001627 - 14:27
    f60c22ff020101e8320d00163c - 14:28
    f60c22ff020101d0360d001650 - 14:29
    f60c22ff020101b83a0d001664 - 14:30
    f60c22ff02010104420d001610 - 14:29
    f60c22ff020101ec450d001625 - 14:30
    f60c22ff02010006fe0c001654 - Start Time
    f60c22ff02010128b90c001676 - Stop Time
    f60c22ff02010028b90c001677 - Start Time: 13:54
    f60c22ff02010173900c001654 - Stop Time:  13:44
    f60c22ff0201015b940c001668 - Stop Time:  13:45
    f60c22ff02030008f0d6b76b88 - 
    f60c22ff020101dd170c001663 - Stop: 13:13
    f60c22ff02010195090c0016b9 - Stop: 13:09
    f60c22ff02010104e20b001672 - Stop: 12:59          0000001000000001000000010000010011100010000010110000
    f60c22ff0201014ce50b001627 - Stop: 13:00          0000001000000001000000010100110011100101000010110000
    f60c22ff02010134e90b00163b - Stop: 13:01          0000001000000001000000010011010011101001000010110000
    f60c22ff0201011ced0b00164f - Stop: 13:02          0000001000000001000000010001110011101101000010110000          
    f60c22ff02010104f10b001663 - Stop: 13:03          0000001000000001000000010000010011110001000010110000
    f60c22ff020600004845415453 - Change Home team name
    f60c22ff02060000484f4d454c - Change Home team name
    f60c22ff020600004845415453 - Change Home team name to HEAT
    f60c22ff02060000484f4d454c - Change Home team name to HOME    
    zzvvttuu..xixa..........yy
    zz = f6 initiator
    yy = checksum?
    vv = Packet Length
    ttuu = Header. It seems Data can exist between the tt and uu? tt also seems to be able to change?
    a  = Time stopped. 0 = Running, 1 = Stopped
    i  = Some sort of uid?

Penalty Info
    f60ddcff02010581c0d4010000a4 - Home 1st pen #4 2:00
    f60ddcff0201058112c201000064 - Home 1st pen #4 1:56
    f60ddcff0201058112c201000064 - Home 1st pen #4 1:56
    f60ddcff020105815cb501000027 - Home 1st pen #4 1:52
    f60ddcff0201058280a90300000c - Home 2nd pen #9 4:00
    f60ddcff02010582ca9c030000cf - Home 2nd pen #9 3:57
    f60ddcff02010583c0d4010000a2 - Away 1st pen #23 2:00
    f60ddcff0201058351c90100001c - Away 1st pen #23 1:58
    f60ddcff02010584c0d4010000a1 - Away 2nd pen #49 2:00
    f60d22ff02010581c0d40100005e - Home 1st pen #4  2:00
    f60d22ff020105812dcf010000f6 - Home 1st pen #4  1:59
    f60d22ff020105813acb010000ed - Home 1st pen #4  1:58
    f60d22ff020104813acb010000ee - Home 1st pen #4  on Start Time
    f60d22ff0201058165c7010000c6 - Home 1st pen #4  1:57
    zzvvttuu....xbxa..........yy
    zz = f6 initiator
    yy = checksum?
    vv = Packet Length
    ttuu = Header. It seems Data can exist between the tt and uu? tt also seems to be able to change?
    a = Scoreboard Pen slot
    b = Time running? 4 = Stopped, 5 = Running 

Triggered when I hit reset
    f613dcff02010b2909120006190800da07000062
    f613dcff02010b170d120006190800da07000070
    f61322ff02010b15000f00041e0800da07000039
    f61322ff02010b200a0f00041e0800da07000024 - Triggered when away pen went up #99 2min

Unknown
    f614dc0000ff0848f21b3b9a0300006e686f6e6e65
    f614dc0000ff0848f21b3b9b0300007068706b7062
    f614dc0000ff0848f21b3b9c0300006d686d5f6d76
    f614dc0000ff0848f21b3b9d030000706872707059
    f614dc0000ff0848f21b3b9e0300006b686b6b6b6e
    f614dc0000ff0848f21b3b9f0300006f686f576f75
    f614dc0000ff0848f21b3ba00300005868585258be
    f614dc0000ff0848f21b3ba10300006e686e506e7d
    f614dc0000ff0848f21b3ba20300005e68725e5e8a
    f614dc0000ff0848f21b3ba30300005a686a5a5a9d

    f614400000ff0848f21b3be10000006f5d6f6f6fc5
    f614400000ff0848f21b3be20000006d5d6d6c6dcd
    f614400000ff0848f21b3be3000000705d706e70c1
    f614400000ff0848f21b3be40000004b5d6a4b4b33
    f614400000ff0848f21b3be50000004d5d724d4d24
    f614400000ff0848f21b3be60000006d5d6d6b6dca
    f614400000ff0848f21b3be70000006b5d6e6b6bcc
    f614400000ff0848f21b3be8000000705d705570d5
    f614400000ff0848f21b3be90000006c5d706c6cc5
    f614400000ff0848f21b3bea0000006d5d706d6dc1
    f614220000ff0848f21b3b5d0100006b006c6b6bd2
    f614220000ff0848f21b3bf70100006d006f6d6d2f
    f614220000ff0848f21b3b0002000057006c57576a
    f614220000ff0848f21b3b0802000056006556566c
    zzvvttdddduu..........aabb..............yy
    zz = f6 initiator
    vv = Packet Length
    yy = checksum?
    ttuu = Header. It seems Data can exist between the tt and uu? tt also seems to be able to change?
    dd = Data between ttuu
    aa = Some sort of counter
    bb = Counter pt 2

Unknown
    f618ffff0004a304c85fdc170148f21b3b486f636b657931aa
    f618ffff0004a304c85f40170148f21b3b486f636b65793146
    f618ffff0004a304c85f22190148f21b3b486f636b65793162
    					              486f636b657931 = Hockey1 which corresponds to the "network"

Generic Scoreboard state?
    f622dcff0200100100000000000000000001010100000000000000ffffffff0000009b
    f622dcff0200103200000000000000000001000000000000000000ffffffff0000006c - 14 min, per 1
    f622dcff0200103300000001000000000001000000000000000000ffffffff0000006a - 14 min, per 1, home 1
    f622dcff0200103400000002000000000001000000000000000000ffffffff00000068 - 14 min, per 1, home 2
    f622dcff0200103500000002010000000001000000000000000000ffffffff00000066 - 14 min, per 1, home 2, away 1
    f622dcff0200103600000002010100000001000000000000000000ffffffff00000064 - 14 min, per 1, home 2, away 1, home shot 1
    f622dcff0200103800000002010201000001000000000000000000ffffffff00000060 - 14 min, per 1, home 2, away 1, home shot 2, away shot 1
    f622dcff0200103900000002010201000001010100000000000000ffffffff0000005d - 13:54 , per 1, home 2, away 1, home shot 2, away shot 1
    f622dcff0200103a00000002010201000002010100000000000000ffffffff0000005b - 13:54 , per 2, home 2, away 1, home shot 2, away shot 1
    f622dcff0200103c00000002010201000002010100000000000000f4ffffff01000063 - 13:54 , per 2, home 2, away 1, home shot 2, away shot 1, home pen 1 indicator
    f622dcff0200103f00000002010201000002010100000000000000f4f9ffff00000067 - 13:49 , per 2, home 2, away 1, home shot 2, away shot 1, home pen 0 indicator
    f622dcff0200104100000002010201000002010100000000000000f4f923ff00010040 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1, away pen 1 indicator
    f622dcff0200104300000002010201000002010100000000000000f4f92349000200f3 - 13:43 , per 2, home 2, away 1, hpme shot 2, away shot 1, away pen 2 indicator
    f622dcff0200104c00000002010201010002010100000000000000ffffffff00000048 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1, home goal blinking
    f622dcff0200104d00000002010201000002010100000000000000ffffffff00000048 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1
    f622dcff0200104e00000002010201000102010100000000000000ffffffff00000046 - 13:43 , per 2, home 2, away 1, home shot 2, away shot 1, away goal blinking
    f62240ff0200100e00000000040000000001000000000000000000ffffffff00000028
    f62222ff020010fe0000001f000000000001010100000000000000ffffffff00000039
    f62222ff020010ff00000020000000000001010100000000000000ffffffff00000037
    f62222ff0200100001000021000000000001010100000000000000ffffffff00000034
    f62222ff0200100101000022000000000001010100000000000000ffffffff00000032
    zzvvttuu......ppqq....aabbccddeeffgg..................hhiijjkkllmm..yy
    zz = f6 initiator
    yy = checksum?
    vv = Packet Length
    ttuu = Header. It seems Data can exist between the tt and uu? tt also seems to be able to change?
    aa = Home score
    bb = Away score
    cc = Home Shots
    dd = Away shots
    ee = Home Goal
    ff = Away goal
    gg = Period
    hh = Home 1st penalty number
    ii = Home 2nd penalty number
    jj = Away 1st penalty number
    kk = Away 2nd penalty number
    ll = Home Penalty Indicator
    mm = Away Penalty Indicator
    pp = Packet counter
    qq = PAcket counter 2?
```
{{< /details >}}

### Time

The one piece of information, outside of the header information, I haven't been able to identify from the packet data is the time on the scoreboard. One of the packets, `f60b22ff020108800e313b79` does show the time of day as `0e313b` equals 14:49:59 or 2:49 pm, but the same can't be said for the other time packets. I've been able to slowly deduce the packets that contain the time info via process of elimination and have logged a few of the packets and the time they correspond to

```
f60c22ff020101781b0d0016c3 - 14:20
f60c22ff02010148230d0016eb - 14:21
f60c22ff020101002f0d001627 - 14:27
f60c22ff020101e8320d00163c - 14:28
f60c22ff020101d0360d001650 - 14:29
f60c22ff020101b83a0d001664 - 14:30
f60c22ff02010104420d001610 - 14:29
f60c22ff020101ec450d001625 - 14:30
```

From this I can reduce the noise based upon what I know is already decodable and the bits that don't change to get

```
781b0d0016 - 14:20
48230d0016 - 14:21
002f0d0016 - 14:27
e8320d0016 - 14:28
d0360d0016 - 14:29
b83a0d0016 - 14:30
04420d0016 - 14:29
ec450d0016 - 14:30
```

Somehow these 5 bytes translate to the given time, but I cannot figure out how, especially since `b83a0d0016` and `ec450d0016` both seem to mean 14:30. Additionally the penalty info, also reduced like above gives 

```
c0d401 - 2:00
12c201 - 1:56
5cb501 - 1:52
```

Which is only 3 bytes, but still produces the same issue...



### Checksum

What's up with the check sum? Well, I think it's some sort of check sum or parity calculation. It's at the end of the packets, it changes when the content of the packet changes and stays the same otherwise. I cannot for the life of figure out the algorithm for calculating this and asking ChatGPT just causes it to break simple arithmetic to get an answer.

![Image bad AI math](./ai_math.png)

### ttuu?

Now that ttuu business. I'm not really sure here. Initially I thought it was some sort of extended header, but then I started leaning to some sort of source & destination info, but I'm still not completely positive. The next 2 bytes may also be lumped into this, but again, I don't have enough information to know for sure. On thing I do know is that the tt value can change & does change each time the controller is started. I don't know why this is, or where this value comes from, but all the ttuu values that are `dcff` one time are `40ff` the next time and restarting again they become `22ff` or another byte. It's easy to spot, and as far as I can tell stays consistent during the normal operation of the controller. These aren't the only values of ttuu though, just the most common. I'll take a deeper dive on this soon.


### Packet ID
Finding the packet ID is a very crucial piece of information as it makes sorting, filtering, and decoding packets a world of a difference. Without it, I am currently relying on the packet length to know what packet is what. This obviously isn't ideal as multiple different packet types could have the same length. Going back to the partial packet data mask I deduced above `zzvvttuu......ppqq....aabbccddeeffgg..................hhiijjkkllmm..yy` Assuming these packets do have ID's (I'd be shocked if they didn't), There are only a few places the ID could live. The empty space between gg and hh doesn't make sense to have an ID as that is squarely within the core "data" of the packet and just wouldn't be realistic to put the ID there. Similarly this is the case for the 2 bytes between qq and aa. This leaves us with 4 bytes. 3 of which directly follow the unknown ttuu bytes and 1 at the end right before the suspected checksum. The end is possible, but unlikely and other packets do have decoded data in that spot, so that is also out leaving just the 3 bytes at the start of the packet after ttuu.

Looking at the packet that the above mask goes with, I see that `dcff` is out ttuu value and `020010` is the unknown 3 bytes. Lets look at the other packets and compare them all below:

```
dcff    020010  - Scoreboard Info
dcff    020108  - Time of day related
dcff    020107  - Penalty less info
22ff    020101  - Time?
22ff    020100  - Time? (When switch flipped to start running the time)
ffff    0004a3  - Unknown something with the network name in it
2200    00ff08  - Unknown
```

You can see that this is from 2 separate sessions as the tt value is different, but the point to notice is that all the time based packets seem to have a `0201` first 2 bytes and the next byte changes, but it also changes with some sort of state which doesn't feel like it'd be a packet ID?

The last packet in this list though just ruins everything. `00ff` just feels like an extension of the ttuu we've been seeing, but the 08 makes the time of day packet... which invalidates that being the packet ID? This is where My theory of ttuu being bundled with the the next 2 bytes into some sort of source & destination originated from and that I'll get into next.

### Start-up Sequence

Let's take a step back and start from the beginning. Up till now I've been looking at the packets as they come in midway through the use of the scoreboard and controller. Maybe recording all the packets as they come in from the controller right after I turn it on will help me deduce some of the packet structure. Specifically the ttuu funky business and maybe even the packet ID. I've broken up the packets visually below into a packet number and then the header, length, ttuu, next 2 bytes, next byte, rest of the data for easier analysis.

{{< details summary="Packets on startup" >}}
```
===== Start Up =====
1)  f6  06  ffff    0000    a6
2)  f6  06  ffff    0000    a6
3)  f6  06  ffff    0000    a6
4)  f6  30  ffff    00ff    02  48f21b3b8a3700000031323833000000000000000000000000000000000000000000000000000000005c
5)  f6  0a  fffd    0003    48  f21b3b11
6)  f6  0c  fffd    0204    48  f21b3b01ff0c
7)  f6  0d  ffff    02ff    04  010031323833cb
8)  f6  43  ffff    01ff    00  8001660000008901090000008a01370000004a01010000008b013a0000004b01010000008c01010000004c01010000008d011a0000004d0101000000fb
9)  f6  0a  fffd    0010    48  f21b3b04
10) f6  62  ffff    00ff    10  00000068d4df2a000000000100000008000000486f636b6579000000000000000000000000000000000000000000000000000048f21b3b8a0000003132383300000000000000000000000000000000000000000000000000000000a2
11) f6  62  ffff    00ff    11  000100a304c85f000000000100000008000000486f636b6579310000000000000000000000000000000000000000000000000048f21b3b8a0000003132383300000000000000000000000000000000000000000000000000000000e6
12) f6  62  ffff    00ff    11  00020028160a56000000000100000008000000486f636b6579320000000000000000000000000000000000000000000000000048f21b3b8a000000313238330000000000000000000000000000000000000000000000000000000014
13) f6  0a  ffff    00ff    11  0003008f
14) f6  0c  fffd    0203    48  f21b3b02ff0c
15) f6  1a  ffff    02ff    03  020008f0d6b76ba304c85f8e3cbd694fea9611fe

===== Select Network =====
16) f6  06  ffff    0000    a6  
17) f6  06  ffff    0000    a6  
18) f6  06  ffff    0000    a6  
19) f6  30  ffff    00ff    02  48f21b3b8a3700000031323833000000000000000000000000000000000000000000000000000000005c
20) f6  0a  fffd    0003    48  f21b3b11
21) f6  0c  fffd    0204    48  f21b3b01ff0c
22) f6  0d  ffff    02ff    04  010031323833cb
23) f6  0a  2901    0018    a3  04c85f90
24) f6  de  29ff    00ff    18  000003000000a304c85f39000000e002000047414d454f505453040000000e000000000000030100000300000003000000030000000300000003000000030000000300000003000000030000000300000003000000030000000347414d4554494d450c0000000e00000000000000e0322900e03229000000000000530700407e050000000000e0930400e093040000000000c0d40100c0d4010000000000e0322900e03229000000000080fc0a0080fc0a000000000080fc0a0080fc0a0000000000804f1200804f12000000000040d10c00c0d4010000f4
25) f6  de  29ff    00ff    19  0001000000000000000000000000000000000000000000000000000000804f1200804f120000000000000000000000000000000000000000000000000054494d454f555420040000000e00000060ea000060ea000060ea000060ea000060ea000060ea000060ea000060ea000060ea000060ea000060ea000060ea000060ea000060ea00004241534b42414c4c3400000001000000b8880000204e00000000000000000000070000000a0000000a0000000400000003000000307500000000000000000000983a0000464f4f5442414c4c18000000010040
26) f6  de  29ff    00ff    19  00020000409c0000a8610000000000000300000000000000000000004241534542414c4c0c00000001000000204e00003075000000000000534f43434552202008000000010000000200000000000000564f4c5942414c4c040000000100000000000000484f434b45592020080000000100000000000000c0d401004c4143524f5353451000000001000000905f010060ea00000000000000000000474f414c4c47485404000000010000000000000053484f54434c434b08000000010000000000000000000000484f4d454e414d45100000000a000071
27) f6  de  29ff    00ff    19  00030048454154000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000474553544e414d45100000000a000000000000000000000000000000000000000000000000000000000000000000000000000000f9
28) f6  85  29ff    00ff    19  000400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e1
29) f6  06  29ff    0a05    6d  
30) f6  08  29ff    0201    07  81ef
31) f6  08  29ff    0201    07  82ee
32) f6  08  29ff    0201    07  83ed
33) f6  08  29ff    0201    07  84ec
34) f6  0c  29ff    0203    00  0828fafe6bde
35) f6  08  29ff    0201    07  81ef
36) f6  08  29ff    0201    07  82ee
37) f6  08  29ff    0201    07  83ed
38) f6  08  29ff    0201    07  84ec
39) f6  0c  29ff    0201    01  40d10c00163f
40) f6  08  29ff    0201    07  81ef
41) f6  08  29ff    0201    07  82ee
42) f6  08  29ff    0201    07  83ed
43) f6  08  29ff    0201    07  84ec
44) f6  13  29ff    0201    0b  19280f00041e0800da07000006
45) f6  0c  29ff    0203    00  0828fafe6bde
46) f6  0c  29ff    0201    01  40d10c00163f
```
{{< /details >}}

I can see here that prior to selecting a network, the "normal" tt value that ends up being 29 is not seen at all. Instead I see only ff and some sporadic fd. Analyzing the rest of the packet data I can see a few things

```
4) Contains the hex '31323833' which is ASCII '1283' which corresponds to the model scoreboard at the arena
7) Again contains the hex '31323833'
10) Contains the hex '486f636b6579' which is ASCII 'Hockey' and again has the scoreboard model hex '31323833'
11) Contains the hex '486f636b657931' which is ASCII 'Hockey1' and again has the scoreboard model hex '31323833'
12) Contains the hex '486f636b657932' which is ASCII 'Hockey2' and again has the scoreboard model hex '31323833'
13) Has nothing, but looking at this back and the previous 3, I see that start with the following hexes in order, '0000', '0001', '0002', '0003' which seems like some sort of counter or indicator. Important soon
```

Once I select a network on the controller I can start to see the `29ff` become common place for the ttuu values, but one thing to note is that `2901` is also seen. Looking at the next too bytes I see a lot of `0201`, `0203`, `00ff`, `0000`, essentially `00`, `01`, `02`, `03`, or `ff` byte components. This started to make me think it was something to do with source and destination and then something network related as 0,1,2, and 3 are all noted in that packet sequence from start up and `ff` could correspond to a broadcast meaning any network. This breaks down slightly as I see some other bytes like `18`, `0a`, `04`, `05`. I think though that there's too much noise here. I'm seeing packets sent by both the controller and the scoreboard and I'm not able to differentiate which packet comes from where. Let's try this again , but this time not turn the scoreboard on, only the controller, so I only get the controllers initial packets.