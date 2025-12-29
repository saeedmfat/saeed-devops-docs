# 50 Important iptables Commands and Tricks

## Basic Management
1. **`iptables -L`** - List all rules
2. **`iptables -L -v`** - List rules with packet counts
3. **`iptables -F`** - Flush all rules (delete)
4. **`iptables -X`** - Delete all custom chains
5. **`iptables -Z`** - Zero packet counters
6. **`iptables-save > file.txt`** - Backup rules to file
7. **`iptables-restore < file.txt`** - Restore rules from file

## Chain Operations
8. **`iptables -P INPUT DROP`** - Set default policy to DROP
9. **`iptables -P FORWARD ACCEPT`** - Set forward policy
10. **`iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT`** - Append rule
11. **`iptables -I INPUT 1 -s 192.168.1.10 -j DROP`** - Insert rule at position
12. **`iptables -D INPUT 3`** - Delete rule number 3
13. **`iptables -N MYCHAIN`** - Create custom chain
14. **`iptables -X MYCHAIN`** - Delete custom chain

## Filtering by IP/Network
15. **`iptables -A INPUT -s 192.168.1.1 -j DROP`** - Block specific IP
16. **`iptables -A INPUT -d 192.168.1.0/24 -j ACCEPT`** - Allow destination network
17. **`iptables -A INPUT -s 192.168.1.0/24 -j DROP`** - Block entire subnet
18. **`iptables -A INPUT ! -s 192.168.1.0/24 -j DROP`** - Allow only local network

## Port and Protocol Filtering
19. **`iptables -A INPUT -p tcp --dport 22 -j ACCEPT`** - Allow SSH
20. **`iptables -A INPUT -p tcp --dport 80 -j ACCEPT`** - Allow HTTP
21. **`iptables -A INPUT -p tcp --dport 443 -j ACCEPT`** - Allow HTTPS
22. **`iptables -A INPUT -p tcp --dport 80:100 -j ACCEPT`** - Allow port range
23. **`iptables -A INPUT -p udp --dport 53 -j ACCEPT`** - Allow DNS
24. **`iptables -A INPUT -p icmp -j ACCEPT`** - Allow ping
25. **`iptables -A INPUT -p tcp ! --syn -j ACCEPT`** - Allow established connections

## Interface-Based Rules
26. **`iptables -A INPUT -i eth0 -j ACCEPT`** - Allow input on interface
27. **`iptables -A OUTPUT -o eth0 -j ACCEPT`** - Allow output on interface
28. **`iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT`** - Forward between interfaces
29. **`iptables -A INPUT -i lo -j ACCEPT`** - Allow loopback

## Stateful Filtering
30. **`iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`** - Allow established
31. **`iptables -A INPUT -m state --state NEW -p tcp --dport 22 -j ACCEPT`** - Allow new SSH
32. **`iptables -A INPUT -m state --state INVALID -j DROP`** - Drop invalid packets

## NAT and Masquerading
33. **`iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`** - Basic NAT
34. **`iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -j SNAT --to 1.2.3.4`** - Source NAT
35. **`iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.1.10:80`** - Port forwarding
36. **`iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE`** - Subnet NAT

## Rate Limiting
37. **`iptables -A INPUT -p tcp --dport 22 -m limit --limit 3/min -j ACCEPT`** - Limit SSH attempts
38. **`iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT`** - Limit HTTP

## Advanced Matching
39. **`iptables -A INPUT -m mac --mac-source 00:0F:EA:91:04:08 -j DROP`** - Block by MAC
40. **`iptables -A INPUT -p tcp --tcp-flags ALL SYN,ACK -j DROP`** - Filter TCP flags
41. **`iptables -A INPUT -m multiport --dports 22,80,443 -j ACCEPT`** - Multiple ports
42. **`iptables -A INPUT -m time --timestart 09:00 --timestop 17:00 -j ACCEPT`** - Time-based rules

## Logging and Debugging
43. **`iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "SSH Attempt: "`** - Log SSH attempts
44. **`iptables -A INPUT -j LOG --log-prefix "Dropped: " --log-level 4`** - Log dropped packets
45. **`iptables -A INPUT -p icmp --icmp-type 8 -j LOG --log-prefix "Ping: "`** - Log ping requests

## Security Hardening
46. **`iptables -A INPUT -p tcp --syn -m connlimit --connlimit-above 10 -j DROP`** - Prevent SYN floods
47. **`iptables -A INPUT -f -j DROP`** - Drop fragmented packets
48. **`iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP`** - Drop XMAS packets
49. **`iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP`** - Drop NULL packets
50. **`iptables -A INPUT -m recent --name SSH --update --seconds 60 --hitcount 5 -j DROP`** - SSH brute force protection

## Quick Tricks:
- **`iptables -L --line-numbers`** - Show rules with line numbers
- **`watch iptables -L -v`** - Monitor rules in real-time
- **`iptables -D INPUT $(iptables -L INPUT --line-numbers | grep 192.168.1.1 | awk '{print $1}')`** - Delete rule by IP
- **`iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`** - Always put this first
- **`iptables -A INPUT -j REJECT --reject-with icmp-host-prohibited`** - Reject with message instead of drop

## Important Notes:
- Rules are evaluated top to bottom
- First match wins
- Default policies apply if no rules match
- Changes are temporary (use iptables-save to make permanent)
- Order matters significantly in rule placement
