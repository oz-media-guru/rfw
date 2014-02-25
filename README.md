rfw - remote firewall
=================================

Firewall with remote control. rfw is the RESTful service which applies iptables rules for individual IP addresses on request from remote client.   
rfw maintains the list of blocked IP addresses which may be updated in real time from many sources. rfw also solves the problem of concurrent modifications to iptables since the requests are serialized. 

Typical use case
---------------------------------
You manage a group of machines which are deployed/controlled/monitored from a central server or admin panel. You need to react quickly/automatically on abuse/DDOS with the rules generated by the intelligence/analytics/geolocation-aware server. You push the IP blocklist updates to other machines in real time.

Features
---------------------------------
- block individual IP addresses on iptables on request from remote host
- serialize requests to prevent concurrency issues with iptables
- remove duplicate entries
- bulk updates in order to sync client blocklist with rfw status - ?? It doesn't seem to be a good idea. Keep it simple, move sync responsibilty to the client. Let the client GET the rules list and PUT the missing ones.
- both remote and local interface
- remote updates via RESTful API
- secured with HTTPS
- authenticated with basic authentication over SSL and by client source IP
- keep IP whitelist (it's better to call it ignored IPs) to prevent locking access to the machine.
- whitelist defined in config file, handle CIDR notation
- act as the iptables guard
- local client with syntax identical to iptables
- when available use ipset for high performance 
- blocking individual IPs with expiry periods
- doesn't interfere with more general iptables rules
- works well with fail2ban
- initialize iptables with static set of rules - ?? not sure if needed
- attacks protection:
    - only enable clients by static IP
    - or open for all, but ban after failed attempts (like fail2ban)


Optional
---------------------------------
- persistent log of events to be replayed after reboot - ?? Maybe just a separate iptables command log


FAQ
---------------------------------
**Q: Why not use chef/puppet/ansible/salt/fabric/ssh instead?**

For a couple of reasons:   
- Security, trust and permission management. The above tools require giving a remote client the ssh root acces. Often we want to allow the IP analytics server to be able to block selected IPs without giving admin rights.
- Performance - handle frequent and concurrent requests
- No dependencies and easy to talk to from any platform and language. RESTful - lingua franca of the Internet
- Protection against locking yourself out by applying erroneous rule

Note that when the rules come from variuos sources they may interact badly. For firewalls the order of rules matters. That's why the functionality of remote rfw is limited to blocking individual IPs inserted in front of the ruleset. Be careful when using local rfwc where you have the full power of iptables at hand. 


**Q: rfw limits REST client access by IP whitelisting. What if I need to connect from dynamic IP?**

rfw is intended for hosts with static IP addresses. It includes both servers and clients. For clients it is not as strong requirement as it seems since in typical rfw deployment the client is a data center collocated machine with static IP. If you really need to use REST client from various locations or from dynamic IP, you have a couple of options:

- If you have any server with static IP with SSH access use it as a gateway client to rfw.
- If you have dynamic IP from particular address pool assigned to your Internet Service Provider you may whitelist the entire address range.
- You can connect through VPN with static IP.

**Q: Is it secure?**

Tampering with core firewall should never be taken lightly. rfw must be run with root-like privileges in order to modify iptables so it requires a lot of trust in the software. 
There is a trade-off 
Sometimes there is no choice and you need to automate firewall actions anyway. While rfw is designed with distributed system in mind, it may improve security even for a single box by:
- limiting iptables functionality to operate only on individual IP addresses
- whitelisting selected IP addresses to prevent lock out
- serializing iptables modification

it provides advantage over changing iptables manually.

Security of rfw was the primary concern from the very beginning and influenced these design decisions:
- simplicity - no fancy features
- no external dependencies except iptables
- limited functionality - no generic rules
- not performance-optimal but conservative choice of time-proven crypto: 2048-bit RSA based SSL with HTTP Basic Authentication



TODO
---------------------------------
- Start with a single argument -f config file. Default /etc/rfw/rfw.conf
- Write config file first to refine requirements
- Logging to syslog. When started log whitelisted (ignored) ips.
- Ignored IP list. The list of IPs which are never applied on iptables. Should HTTP response be different or just log and ignore?
- Ruleset order: blocked ips DROP, ACCEPT ignored IPs [on rfw port], all IPs on rfw port DROP, the rest.
- /etc/rfw/white.list and /etc/rfw/black.list. Locations relative to config file and overridable in config file.
- Asynchronous processing. Single threaded http server pushing requests to the queue. Another iptables guard thread to consume from the queue.
- use PUT and DELETE - should be idempotent. Give the option on non-restful requests with GET and additional ?verb=PUT/DELETE param in query
- Clarify responsibilities of rfw and rfwc
    - rfw need to store credentials in config file so it shouldn't be mixed with rfwc
    - rfw need to take an optional argument -f config_file
    - rfwc should be the lightweight http client submitting jobs to rfw
    - rfwc may take arguments in 2 forms:
        - single line rule to mimic iptables interface
        - multiline bash scripts with iptables commands
    - rfw/rfwc should guarantee executing multiline scripts in order while being uninterrupted by other rfw requests
    - both rfw and rfwc should accept -v/-h options
    - show examples of using rfwc as iptables replacement:
        - as rfwc <iptables args> - should be synchronous to be able to report errors
        - as symbolic link for global iptables substitution - synchronous
        - as rfwc without command line options but reading scripts from standard input - asynchronous
        - as above but with -b batch_file.sh option instead of standard input - asynchronous
        - if you want the 2 above make synchronous i.e. to wait for the rfw to finish processing iptables commands, simply call rfwc again with --wait flag. There should be a special call to rfw that does not return response until the queue is empty.
- rfw config file options: in rfw.conf
- The default use case should be DROP individual IP on INPUT chain. Make the action (DROP/ACCEPT) configurable. It may be useful for FORWARD chain.
- python packaging and release scripts
- documentation
- run as daemon (check fail2ban code)
- intercept SIG_TERM and SIG_INT for graceful termination
- add --non-daemon option at rfw startup
- implement timeouts - check fail2ban

 
REST queries:
---------------------------------
- to modify INPUT chain:  
PUT /input/input_iface/src_ip[?wait=true[&timeout=n_sec]]  
DELETE /input/input_iface/src_ip[?wait=true]  

TODO add wait=true options in description

- to modify OUTPUT chain:
PUT /output/output_iface/dst_ip[?timeout=n_sec]  
DELETE /output/output_iface/dst_ip  
 
- to modify FORWARD chain:
PUT /forward/input_iface[/src_ip[/output_iface[/dst_ip[?timeout=n_sec]]]]  
DELETE /forward/input_iface[/src_ip[/output_iface[/dst_ip]]]  
 
- to list rules:
GET /chain[/iface]  
TODO allow various formats of rules list  

- return help info for client. Response should include server ip, port, and relevant rfw configuration details
GET /  



Examples:
---------------------------------

PUT /input/eth0/12.34.56.78?timeout=3600                ===   iptables -I INPUT -i eth0 -s 12.34.56.78 -j DROP with expiry 3600 seconds

DELETE /input/eth0/12.34.56.78                          ===   iptables -D INPUT -i eth0 -s 12.34.56.78 -j DROP

PUT /input/any/12.34.56.78                              ===   iptables -I INPUT -s 12.34.56.78 -j DROP

DELETE /input/any/12.34.56.78                           ===   iptables -D INPUT -s 12.34.56.78 -j DROP

PUT /output/ppp/12.34.56.67                             ===   iptables -I OUTPUT -i ppp+ -d 12.34.56.78 -j DROP

PUT /forward/ppp/11.22.33.44/eth0/55.66.77.88           ===   iptables -I FORWARD -i ppp+ -s 11.22.33.44 -o eth0 -d 55.66.77.88 -j DROP

PUT /forward/any/0.0.0.0/any/55.66.77.88                ===   iptables -I FORWARD -d 55.66.77.88 -j DROP

PUT /forward/tun/11.22.33.44                            ===   iptables -I FORWARD -i tun+ -s 11.22.33.44 -j DROP
 
PUT /input/eth0/12.34.56.78?wait=true                   ===   iptables -I INPUT -i eth0 -s 12.34.56.78 -j DROP    and wait for finishing processing this iptables command (previous request in the queue must also be processed)


 
0.0.0.0 can only be used in FORWARD chain to signal any IP   
iface without number like ppp means ppp+ in iptables parlance  
any in place of interface means any interface  

PUT means for iptables:
- for INPUT chain: insert the rule matching packets with specified source IP and input interface and apply DROP target
- for OUTPUT chain: insert the rule matching packets with specified destination IP and output interface and apply DROP target

DELETE means: DELETE the rule  
PUT checks for duplicates first so subsequent updates do not add new rules, but it is not purely idempotent since it may update the expiry timeout  




Design choices
---------------------------------

Note that HTTPS is not the perfect choice protocol here since by default it authenticates the server while we need to authenticate the client. Anyway we want to use standard protocols here so we stick to the SSL + basic authentication scheme commonly used on the web. SSL authenticates the server with certificates while shared username + password authenticates the client. Client certificates in HTTPS are possible but not all client libraries support it; also it would complicate deployment. 



Testing with curl:  
curl -v --insecure --user mietek:passwd https://localhost:8443/input/eth/3.4.5.6



License
---------------------------------
Copyright (c) 2014 [SecurityKISS Ltd](http://www.securitykiss.com), released under the [MIT license](LICENSE.txt).   
Yes, Mr patent attorney, you have nothing to do here. Find a decent job instead.  
Fight intellectual "property".

