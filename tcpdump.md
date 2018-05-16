to capture one packet NTP traffic
```
sudo tcpdump -ennv -i vtnet0 -c 1 port ntp
sudo tcpdump -ennv -i vtnet0 -c 1 ip6 and port ntp
```
to capture arp traffic
```
sudo tcpdump 'ether proto 0x0806'
sudo tcpdump arp
```
other techniques
```
sudo tcpdump 'ether host 00:00:24:CE:7B:F8'
sudo tcpdump -s 1536 -X port 80 > /tmp/junk.dump
sudo tcpdump -s 1536 -v host waimea > /tmp/junk.dump
sudo tcpdump -s 1536 -X port sieve -i lo > /tmp/junk.tcp
```
troubleshooting VNC:
```
sudo tcpdump -s 1550 -w /tmp/junk.fvwm.trc port 5904 and host cake
```
http://www.faqs.org/rfcs/rfc1323.html

(TSval) contains the current value of the timestamp clock of the TCP sending the option.

The Timestamp Echo Reply field (TSecr) is only valid if the ACK bit is set in the TCP header; if it is valid, it echos a times- tamp value that was sent by the remote TCP in the TSval field of a Timestamps option.  When TSecr is not valid, its value must be zero.  The TSecr value will generally be from the most recent Timestamp option that was received; however, there are exceptions that are explained below.

on Robert, save 20 x 10MB files
```
sudo tcpdump -s 1536 -C 10 -W 20 -w /tmp/junk.tcp host 173.160.50.209
```
on pine
```
sudo tcpdump -s 1536 -C 10 -W 20 -w /tmp/junk.tcp host 69.136.68.166
```
to screen out a lot of chatter
```
sudo tcpdump -n -vv ! port 137 and ! port 138 and ! arp and ! port 17500 and ! port 5001 and ! stp
```
to screen out bcast & mcast
```
sudo tcpdump -i en2 -lnvv '! ether broadcast and ip[16] < 224'
```
To capture IPv6 router advertisements:
```
sudo tcpdump -vvvv -ttt -i en0 icmp6 and 'ip6[40] = 134'
```
