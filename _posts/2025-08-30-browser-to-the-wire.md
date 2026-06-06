---
layout: post
title: "From the Browser To The Wire: What happens when you press enter"
date: 2025-10-23
categories:
  - networking
  - cybersec
---
<style> body { background: #1e1e1e; color: #e0e0e0; } </style>

# Intro
this post will be going over the full networking process of entering a url, all the way from DNS resolution to low level routing details
im using this post more as a way for me to consolidate all of the knowledge i have learned from my networking studies, so if anything is wrong or misinterpreted please let me know

overview
   - [DNS Process](#dns-process)
   - [CDN Overview](#content-delivery-network)
   - [TCP Handshake](#tcp-handshake)
   - [TLS Handshake](#tls-handshake)
   - [HTTP & Encapsulation](#http-&-encapsulation)
   - [Routing](#routing)


## DNS Process
Domain name resolution protocol is a widely used protocol, its main goal is to convert human readable domains like "Facebook.com" into IP addresses
this way instead of trying to remember "23.24.25.36" we can simply remember "Facebook.com"
DNS runs over port 53/udp but can use TCP mainly for 
- Zone transfer(copying records between dns servers)   
- when the response is to large to fit into a single UDP packet
- DNS over TLS(DoT), DNS over HTTPS(DoH)


DNS is also used for much more then just storing servers IPs its also stores other records such as
- AAAA(stores IPv6 addresses)
- DKIM(stores the public key, used to verify the sender is who they say they are)   
- DMARC(tells the receiver what to do if there’s something wrong)   
- SPF(lists the valid servers that can send emails)   
- CNAMES(holds the name of another domain, used for forwarding)   
- MX(holds the host names of mailservers) 
- PTR(used for reverse DNS lookups, stores the domain name instead of IP)

the General flow of  the resolution process is as follows.
First we enter Facebook.com into a url bar
our browser will now start the DNS process, which has 5 main steps

1. First the browser checks the systems local host files, this is a file on disk(/etc/hosts on linux, C:\Windows\System32\Drivers\etc\hosts on windows), this file stores hardcoded domain to IP mappings, it takes precedence over all else, its commonly used inside LANs to allow for static entrys of domain names.   
2. Next it will check the browsers Local cache, all browsers keep a cache of recently converted domain names, this way we dont need to intiate requests everytime we want to visit a website we commonly use   
3. If not found in the browser cache next we check the systems Local Cache, our system also keeps a cache of recently converted domain names, it works the same as above   
4. If the IP wasnt found in any of the above we now intiate the outbound DNS process, our system constructs a DNS packet and sends it the primary DNS server in our systems network configuration(Learned via DHCP or manually entered).
5. the DNS server receives this DNS request, this server is known as a recursive resolver, meaning it takes over and handles the rest of the resolution for us
    - First the resolver reaches out to a root dns server, there are 13 of these servers(a.root-servers.net to m.root-servers.net) but they are distributed across thousands of nodes worldwide and use anycast routing(see CDN section for anycast explanation), root servers handle refering us to the correct TLD server that handle our domain(.com, .cloud etc)
    - the resolver then recevies the IP of the correct TLD server from our domain and then sends the DNS request to it, TLD servers handle looking up our domain and returning us a list of authoratative nameservers
    - finally the resolver receives the list of Authoritative nameservers, it then chooses one and sends the final DNS request to it, the Authoritative nameserver then handles returning us the value we need in most cases its the A record which stores the IPv4 address

Do Note that all of the above servers keep their own cache of recently converted domains and at each step it checks the cache first, most requests for big sites such as google.com, facebook.com etc will stop at the resolver, just didnt mention the cache at each server for brevity

# Content Delivery Network
Facebook and almost all large sites these days use a CDN so ill quickly explain it
A content delivery network is a global network of distributed servers that cache commonly requested static(images, js, html etc) resources, this means that instead of connecting to a server in let’s say Japan we connect to the closest edge server near us that returns the the resource for us(if it doesn’t have the resource or it’s a non-cachable resource the edge server reaches out to the origin server and pulls it. It does this through any cast routing, any cast routing is a routing method where multiple servers worldwide all broadcast the same IP address, then when the IP broadcast to the internet via BGP ISPs will automatically choose the closest server(closest in the sense of the least hops away not geographically closest but it usually is), this also has the added bonus that users never directly interact with the origin server, giving us a little bit of security 


## TCP Handshake
Now that we have the IP of the server(or edge server whatever) we can start communicating with it, but since most of the web uses TCP as its transport method we need to form a connection with it, this is what the tcp handshake handles its used to setup a reliable and steady connection with the server and make sure both sides are ready too send and receive data between each other, it has 3 main steps
1. First we send a TCP packet with the SYN flag set, this message also includes other data such as
   - MSS: the Maximum segment size defines the largest size in bytes the application data can be inside a single tcp segment, its calculated as MTU - tcp headers - IP headers, which on most machines will result in a MSS of 1460, MSS was built specifaclly to help stop fragmentation at the IP, the MSS is stored in the OPTIONS field of the tcp packet and is a 4 byte value
        - kind (1 byte): 2 → indicates Maximum Segment Size option
        - Length (1 byte): 4 → total length of the option in bytes (always 4)
        - MSS value (2 bytes): the actual maximum segment size in bytes

   - SACK: if the initiator of the connection supports SACK they will include a SACK permitted message, SACK is an extension to TCP and allows for more robust packet loss detection and handling, imagine during communication if the receiver received segments 1 2 4 5 6, its missing segment 3 so it keeps replying back to the sender with ACK 3 until the sender relises, the sender must now retransmit segment 3, 4 and 5, which is inefficient, SACK solves this by allowing the receiver to specify exactly what segment was lost and which ones they recevied, meaning the sender only has to resend the lost one, the SACK negotiation is also stored in the TCP options
        - Kind = 4 (SACK Permitted)
        - Length = 2
   - Timestamp: another very common tcp option that is seen in the TCP handshake are timestamps, timestamps serve multiple purposes such as
        - RTT calculation, timestamps allow for milliseconds accurate RTT calculation as all packets include the timestamp of when it was sent, the receiver then can just do TSecr_echoed - current_time = RTT
        - detecting duplicate/stale packets
     the way it works is that if the sender has timestamps enabled they will include 2 values in the tcp option, TSval which is the current timestamp and TSecr which is 0 for now since its used to echo back the senders TSval 

2. the server receives this SYN packet and if its open to communicate the server will send back a SYN-ACK tcp message, this message also includes other data such as
   - SACK perm: if the server also supports SACK they will include it in their response, this way both sides now agree to use SACK for packet loss/handling
   - MSS: the server will also include their own MSS, the smallest MSS value between them is chosen, but do note that this isnt static, either side of the connection can individually update their own MSS if something like PMTUD or some other form of it runs and detects a smaller value
   - timestamp: if the server has timestamps enabled the SYN-ACK packet will also include a timestamp, TSval will be the timestamp of when the server replys and TSecr will echo the TSval the client sent in the SYN packet

3. finally the client will send back a ACK packet, this final packet finalises the tcp handshake and the connection is now established 

note that there are more tcp options such as fast open, window scaling etc i just didn't add them for brevity
also not all these options are present in every handshake, but they are still very common and enabled on most machines
during the process each side sets up their intial sequence numbers, sequence numbers are used by tcp for segment re ordering, packet loss detection, keeping track of data etc

## TLS Handshake

Once the TCP connection is established, we need to set up a secure encrypted connection. This is done via the TLS handshake. The flow below describes TLS 1.3, which is what modern browsers use, it completes in a single round trip unlike the older 1.2 which needed two.

1. **Client Hello:** the client kicks off the handshake by sending:
   - Supported cipher suites
   - Supported TLS versions
   - A key share: the client already generates its ephemeral ECDHE public key and sends it now, guessing the group the server will pick
   - Client random (32-byte value that feeds into the key schedule)

2. **Server Hello:** the server responds, and in 1.3 this single flight contains almost everything:
   - Chosen cipher suite and TLS version
   - Server random
   - The server's own ephemeral ECDHE key share

   At this point both sides have each other's key shares, so they independently derive the shared secret via ECDHE. Everything after this in the handshake is already encrypted.

   The server then sends, now encrypted:
   - Its certificate (for authentication)
   - A **CertificateVerify** message: the server signs a hash of the handshake so far to prove it actually owns the private key for that certificate. This signature is where RSA or ECDSA comes in, 1.3 uses these for authentication only, never for key exchange
   - A **Finished** message containing a hash of all previous handshake messages

3. **Validation:** the client validates the certificate chain and verifies the server's CertificateVerify signature and Finished message. If anything fails here the connection is dropped.

4. **Client Finished:** the client sends its own **Finished** message. The handshake is now complete and the encrypted session is established.

**Note:** the big change from 1.2 is that the key exchange method is fixed (ephemeral ECDHE) rather than negotiated, and the key share is sent in the very first messages instead of after. This is what lets 1.3 finish in one round trip. It also enables 0-RTT resumption, where a returning client can send data in its very first message, though 0-RTT has replay-attack tradeoffs and isn't always enabled.## HTTP & Encapsulation
<details>
<summary>NOTE: about HTTP/3</summary>
<p>
HTTP/3 uses QUIC, a modern transport protocol built on top of UDP instead of TCP like previous HTTP versions, QUIC provides the reliability and congestion control of TCP but without some of the overhead. It also includes built-in TLS, supports 0-RTT handshakes, enables better multiplexing through individual streams, and solves the head-of-line blocking issue that TCP can face.
   For simplicity, this blog focuses on HTTP/1.1–2, which uses TCP as its transport method.
   If you would like to read more about [QUIC](https://www.auvik.com/franklyit/blog/what-is-quic-protocol/)
</p>
</details>

Now that the secure TLS connection is established, our browser can finally request resources from the server using HTTP (Hypertext Transfer Protocol). HTTP is a Layer 7 application protocol that powers the web. It allows us to request files, submit forms, manage sessions, and interact with web applications.
HTTP uses methods to define the operaiton we would like to do to such as:
   - GET: request a resource
   - POST: upload a resource(such as files)
   - PUT: update a resource(such as usrr detsils)
   - DELETE: delete a resource

for us we would like to request a resource so we senda GET request, http also uses headers which allow us to send other details to the server some common ones are
   - User-Agent: a string thay identifies our device type, such as a mobile phone, desktop etc, it also includes whag browser we are using, the version and more
   - Accept: defines the type of data we will accept, such as html, css etc
   - Cookie: a small string that allows the server to identify us across requests, because HTTP is stateless cookies allow the server to remember us, they are automatically included in requests by the browser(depening on the cookies settings and requests origin) and is what allows us to login once then return later and stay logged in
   - Host: specfies the domain name of the server, required in 1.1 requests

so for examples our request to facebook.com will look like
   
```
GET /index.html HTTP/1.1
Host: facebook.com
User-Agent: <whatever>
Accept: text/html, image/jpeg
cookie: <whatever>
```

Now our browser constructs the GET requests and the encapsulation process happens now
i will be using the OSI model for this explanation so i can explain each layer more in depth, but the modern internet uses the TCP/IP model which merges layers 7,6 and 5 into a single layer

Now the encapsulation process takes place, the above was layer 7 application

1. Next it gets passed down to layer 6 presentation, this layer handles the presentation of the data, stuff such as encoding, compression and encryption etc, here any dangerous characters in the url get to encoded according the url encoding spec, and then the GET requests is encrypted with session  using the key that was created during the TLS handshake

2. Next its passed down the transport layer, this layer handles the end to end communication of data, re ordering, segmentation, packet loss detection etc, here the data is encapsulated into a segment and a TCP header is added this includes;
    - Source port
    - Destination port
    - checksum(used for corrupt/tampered segment detection)
    - TCP flags if any(psh, urg etc)
    - sequence number
    - ACK number(next byte we exepct from server)
    - Window size(how much free space is in our buffer, used by the server for flow control)


3. Next its passed down to the IP layer, this layer handles the routing of data, fragmentation, reassembly etc, at this layer the tcp segment is again encapsulated into a IP packet and an IP header is added onto it which includes
    - Source IP(our private IP for now)
    - destination IP(servers public IPv4)
    - checksum(same as above)
    - need fragmentation flag(only needed if total size of packet is larger then systems MTU, usually avoided at all costs)
    - version(IP version)
    - protocol(protocol in use, TCP for us)
    - TTL(used to stop routing loops, gets decremented at every hop, if hits 0 thr packet is dropped)
    - and more

Before the network stack can add the next header(ethernet) it needs to know destination MAC address, it does this by  checking if the destination IP is in the local network or if it needs to be routed to the default gateway, it does this through an AND operation 
   - the AND operation works by taking the destination IP and subnet mask of the network and comparing the result against the LANs network IP for example
        - subnet mask = 255.255.255.0
        - destination IP = 47.58.29.50
        - we then do 255.255.255.0 AND 47.58.29.50 = 47.58.29.0
        - this does NOT match the network IP(192.168.1.0) so we know we must forward the request to the default gateway

After it knows whether its remote or local our system then uses ARP to find the MAC address of the next hop
Address resolution protocol is a layer 2 protocol used internally inside a LAN to map IPs to MACs, its specifically for IPv4 as IPv6 uses neighbourhood discovery protocol.  
   - First it checks its ARP table, this is an in memory table that stores the recently mapped IPs to MACs, each entry has a TTL(platform dependent, windows is 2 minutes) so it constanly has to be refreshed and updated(unless we create static entrys)
   - if the MAC address is not found we then send a broadcast(destination mac: FF:FF:FF:FF:FF:FF) ARP message to the network asking "Who is 192.168.1.1?"
   - All devices on the network recevie this but all of them drop it apart from the host with the specifed IP, this device then replies back "192.168.1.1 is at aa:bb:cc:dd:ee:ff"
   - the original sender receives this and adds the entry into their ARP table for future use

4. Next its passed down to the Data link layer, this layer handles the Hop-by-hop of data, basic corruption checks etc, here the IP packet is encapsulated into a ethernet frame and a etherner header is added whjch includes:
    - source MAC(Our MAC address)
    - Destination MAC(one we learned via ARP)
    - FCS(used for error detection,its a checksum that's calculated ober the etherner header + payload)
    - EtherType(specifys the encapsulated protocol, IPv4, IPv6, ARP etc)
    - VLAN ID(not always)

NOTE:
   if this was HTTP/3 using QUIC the transport layer would instead apply a UDP header, which is much simplier then a TCP header

## Routing
NOTE:
    the flow below assumes this router is the internet gateway and is NAT enabled, but do note that a devices default gateway isnt always a router, its common in large networks for each vlans default gateway to be a L3 Switch or another L3 device
Now that our packet is fully constructed the network stack passes it to our NICs driver, if its a wireless driver it strips the ethernet header and adds a 802.11 header, it then encrypts the entire packet and forwards it to our gateway
the gateway receives it and then does a series of steps, in order too route it to the next hop in the chain

1. the router receives the packet
2. it then validates the Ethernets FCS is correct, drops it if not
3. it then checks to see if the destination MAC is its own MAC, broadcast MAC or a multicast MAC, drops if not
4. de encapsulates the Ethernet header to access the IP header
5. validates the IP header checksum, drops if incorrect as it cant be trusted
6. the router then consults its routing table to know what to do next, if it finds no valid route and their isnt a default route it will drop the packet

<details>
<summary>Routing table?</summary>
<p>
routing tables are an in memory(usually) mapping, it defines where data should be sent next using the IP and subnet, usually house hold routers will just have static routing tables but enterprise routers inside ISPs and other large networks have dynamic tables and use interior gateway protocols such as OSPF, EIGRP, IS-IS to handle dynamically updating them. routing tables usually consist of multiple entrys and each entry has stuff such as:
</p>
<ul>
<li>destination: IP address to match.</li>
<li>gateway: the next hop IP address to forward the data too, typically 0.0.0.0 means the destination is directly connected</li>
<li>subnet mask: defines the range of IPs the route matchs</li>
<li>metric: a cost value used by routers to pick a route, if multiple routes match it will choose the one with the least cost</li>
<li>interface: the physical/logical interface to send the data out of</li>
</ul>
</details>

7. since the data needs to be routed outside of the network NAT takes place, the router will update the source IP to be its public IPv4(and update the source port to something unique if PAT is in use) and create a mapping inside its NAT table, for example heres a mapping from my routers dashboard

<details>
<summary>NAT?</summary>
<p>
Network address translation is a method used to allow multiple devices inside a LAN which have unique non routable private IPv4 addresses to all share a single public IPv4 address, this was created to address the IPv4 exhaustion problem(IPv6 solves this issue but adoption is slower then expected), the way it works is that when a device wants to send data outside of the network the router will update the source IP to be the routers public facing IPv4 address, it will then update its NAT table to map the outbound connection to the internal private IP that intiated it, there are also different types of NAT such as
</p>
<ul>
<li>**Port Address Translation**: The most common form of NAT for home users and SMBs. It rewrites both the source IP and assigns a unique source port for each connection. This allows many internal hosts to share a single public IP while avoiding collisions at the public IP/port level. PAT is also called NAT overload</li>
<li>**Carrier Grade NAT**: carrier grade NAT is a form of nat where not only your home network all shares a single IPv4 but instead you share IPs with multiple other customers of your ISP, instead of the NAT/PAT process taking place at your router your data is first sent through to your ISPs network and then NAT/PAT takes place at their edge routers, this was created to help conserve public IPv4s even further</li>
</ul>
</details>

```
protocol = tcp	6	
ID = 119473	
State = ESTABLISHED	
// internal -> external
source address = 192.168.1.2	
destination address = 35.190.43.134	
source port = 63602	
destination port = 443	
// external -> internal 
source address = 35.190.43.134	
destination address = 61.xx.xx.xx
source port = 443	
destination port = 63602	
```

8. next the router will update the TTL(TTL is a 1-byte field in the IP header that gets decremented by 1 each hop, if its hits 0 the packet is dropped, its used to help stop routing loops) in the IP header to be current TTL - 1, if its 0 after decrementing the router will drop and potentially send back a icmp "Time Exceeded" message
9. the router will then recalculate the IP header checksum and encapsulate the packet into an ethernet frame
10. it will then update the source MAC to be its own MAC and the destination to be the next hop(learned via the routing table and ARP)

this process repeats(apart from NAT process) for each hop across my ISPs network, potentially passed off to other ISPs until it reaches the destination 
once the data reaches the webserver the process happens in reverse, the webserver de encapsulates the packet to be able to retrieve the application data(our GET request), it will then parse the request, retrieve the request resource and then encapsulate back into an ethernet frame through the process above
the data is then routed across the internet until it hits your router

now the NAT process happens in reverse 
1. the router will validate source IP and source port match the NAT mapping(if port restricted cone NAT), it then looks at the internal private IP and port inside rhe mapping
2. it then updates the destination IP to our private IP and destination port to be our internal port
3. the data is then forwarded over whatever medium(etherner, wifi etc) and our device receives it
4. our device then de encapsulates it, grabs the application payload 
5. our browser then renders the page, runs the JS and potentially sends even more GET request if other files are needed such as css, images etc

## Outro
I hope you enjoyed my first blog post and maybe learned something new about what happens when you press Enter. The journey from your browser to the server involves many layers of protocols, security checks, and routing decisions, and this post only scratches the surface. I’d love to hear your thoughts, corrections, or questions.
