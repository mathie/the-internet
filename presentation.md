footer: © 2015 Graeme Mathieson. CC BY-SA 4.0.
slidenumbers: true

# Type “google.com” into you browser and hit enter
## What happens next?

^ I haven’t done any interviewing for a while but I went through a period of growth in one of the companies I worked for where we were feverishly expanding the development team, so we had to be a little more systematic in our approach to interviewing. Instead of just having an open conversation with candidates to see where it led (which is what I’d previously done in such situations), I wound up preparing a ‘standard’ set of questions. It took a few goes, but eventually I settled on a favourite question for the technical portion of the interview:

^ When I pull up my favourite Internet browser, type “google.com” into the address bar, and press return, what happens?

^ I reckon it’s a doozy of a technical question. There’s so much breadth and depth in that answer. We could talk for hours on how the browser decides whether you’ve entered something which can be turned into a valid URL, or whether it’s intended to be a search term. From there, we can look at URL construction, then deconstruction, to figure out exactly what resource we’re looking for. Then it’s on to name resolution, to figure out who we should be talking to.

^ And then it gets really interesting. We start an HTTP conversation, which is encapsulated in a TCP session which is, in turn, encapsulated in a sequence of IP packets, which are, in turn, encapsulated in packets at the data link layer (some mixture of Ethernet and/or wireless protocols), which – finally! – causes some bits to fly through the air, travel along a copper wire, or become flashes of light through fibre optic.
Now our request emerges – fully formed again, if it survived the journey – at the data centre. The HTTP request is serviced (on which entire bookshelves have been written) and the response follows the same perilous journey back to the browser.

^ The story isn’t over, though. Once the browser has received the response, it still has to interpret it, create an internal representation of the document, apply visual style, and render it in a window. And then there’s the client side JavaScript code to be executed.
It’s an exciting story: one of daring journeys, lost packets, unanswerable questions, and the teamwork of many disjoint routers, distributed across the Internet.

^ The entire story, in all its depth, is way too much to cover in the next 40 minutes, so I’ve picked a few highlights, and I’ll gloss over some other areas. Hopefully it’ll give you some insight into what happens behind the curtain, and maybe it will inspire you to investigate other areas for yourself.

---

# How the Internet works
## Graeme Mathieson
### Email me: [mathie@woss.name][1]
### Tweet me: [@mathie][2]

^ I’m Graeme. I’m a software developer. I’ve spent most of the past decade building web apps with Ruby on Rails, and lately I’ve been playing with iOS development in Swift. But I’ve always had an unhealthy interest in figuring out how the Internet work. I’ve been dabbling in DevOps, sysadmin-ing servers, and wrangling routers for a couple of decades now. I spent my formative years — when I should have been paying attention at school — printing out RFCs on a dot matrix printer and surreptitiously reading them in English class.

---

# google.com ⏎

^ So, we’ve launched our favourite web browser. Our computer has read some bits from the spinning rust, loaded it into main memory, and started executing those bits on the CPU. The browser window opens, ready to accept our bidding. We depress switches on the USB keyboard attached to the computer, which are interpreted as UTF-8 characters, and start to appear in the browser address bar. We hit “enter”. What happens next?

---

# Is it a URL?

* **Yep**. OK, cool, my work here is done.
* **Kinda**. Well, let’s turn it into a well formed URL.
* **Nope**. OK, I’m gonna assume you meant to search for something. Let’s turn it into a well formed URL for a web search.

^ First of all, we have to figure out if the string we’ve entered is a valid URL. If it is, we’re good to go. If not, we’ve got a little bit more work to do. A fully formed URL contains several components, but as human beings, we’re lazy, and we tend to omit some of them for brevity, so the computer has to figure out what we really mean. And modern browsers have combined the search input field and the address bar into a single input field. So we need to figure out whether the user intended to enter a (partial) URL, or a search term. In our case, “google.com” looks like it’s a bit of a URL, so we just need to figure out how to complete it.

---

# HTTP Strict Transport Security

Does this site prefer HTTPS?

* `Strict-Transport-Security` header from a previous request?
* In the browser’s list of HSTS preloads?

^ Secure communication is generally for the win. With HTTP, we specify that a secure connection is required through the scheme in the URL — choosing https instead of http. However, in this case, we’ve omitted the scheme entirely, so how does the browser know whether to prefer a secure connection or not? That’s where HTTP strict transport security comes in. If the browser has previously requested a page from this particular domain, then the response may have included a hint that future requests should default to https. If we’ve never made a request to this particular URL, then the browser checks its built in list of well known domains that prefer https.

---

# HTTP Strict Transport Security

Does this site prefer HTTPS?

* **Yep** OK, set the URL scheme to `https`.
* **Nope** Fine then. If you don’t care for security…

^  “google.com” is in the HSTS preload list, so we’ll default to using https for the scheme.

---

# https://google.com/

^ Now we’ve managed to construct a well formed URL and we can continue to make the request.

---

# Browser cache

Is the URL in the browser cache?

* **Yep** Let’s check it’s still valid.
* **Nope** Well, we’re going to have to fetch it.

^ Web browsers keep a cache of the results of previous requests. The http protocol has well defined semantics for deciding when a particular URL’s response can be cached, how that cache is expired, and whether the validity of cached content needs to be verified.

---

# Browser cache

Is the cached content still valid?

`Expires`

`Cache-Control: max-age`

* **Yep** Awesome. We might skip a network request!
* **Nope** OK, let’s check in with the server.

^ The expires header specifies an absolute timestamp for when a page expires. If we haven’t yet reached that point in time, then the response is probably still valid. If we’ve passed that point in time, then the response might still be valid, but we’ll need to double check. The maximum age specifies a relative time (in seconds) for which a response should be considered valid. If we’ve reached that time, then the response definitely needs to be double checked for validity.

---

# Browser cache

Should the cached content be revalidated?

`Cache-Control: must-revalidate`

* **Yep** OK, let’s check in with the server.
* **Nope** Awesome. Skip to rendering!

^ If the server specified that we must revalidate the response, then we’ll do an HTTP request to check the content, see if it’s still valid. We can take a bit of a shortcut here, and just ask the server if the content has changed. We can figure this out based on the timestamp of the version of the page we have cached and/or based on a tag associated with the page. (This tag is an opaque type — so we can’t take any meaning from it — but it is typically a hash of the dynamic content of the page.)

---

# Parse the URL
 
* **Scheme**: “https”
* **Authority**: “google.com”
* **Path**: “/“

^ At this point, we need to know the scheme (protocol) that we’re attempting to use, and we need to know the authority. Well, really, we just need to know the hostname for now. The authority consists of the user information — a username and, potentially, a password — the hostname, and a port number. In this case, the user information and the port number have been omitted. Since the user information has been omitted, we’re going to attempt to make an unauthenticated — anonymous — request. And without an explicit port number, we’ll look up the well known port number for the https scheme.

---

# DNS Lookup: Browser cache

Is the hostname in the browser’s cache?

* **Yep** Awesome, let’s use that IP address.
* **Nope** OK, we’re going to have to do this the hard way.

^ Modern browsers often maintain their own cache of mappings from hostname to IP address. If they can serve the request from their own cache, it saves the time of going through the system resolver (i.e. a sys call or two). 

---

# DNS Lookup: OS resolver

Is the hostname in the operating system’s cache?

* **Yep** Job done. We’ll use that IP address.
* **Nope** OK, we’re really going to have to look it up.

^ The operating system maintains a global cache of mappings from hostname to IP address. This is shared across all processes for all users on the local computer. If the request can be served from the cache, it will be, and nothing else needs to happen. If not, we’re going to need to dig in a bit further.

---

# Name Service Switch

* Check `/etc/hosts`
* Try multicast DNS
* Perform a DNS lookup

^ The name service switch is a level of indirection inside the operating system’s name service resolution that allows the system administrator to specify a set of resolution mechanisms. In the olden days, this allowed us to insert additional name resolution systems (i.e. NIS/YP) but these days, it’s pretty much just the static `/etc/hosts` file, then falling back on DNS.

---

# DNS Lookup

Get the IP address of a name server

* From DHCP
* Statically configured

^ The IP address of one or more name servers is already known by the operating system. Since you need to have a name server in order to convert hostnames to IP addresses, there needs to be a way to “bootstrap” the process. Typically the IP address of the name servers is supplied by DHCP, or through some other static configuration provided by a system administrator. Most often, the name server will be somewhere on the local network — at home, it’s probably your DSL router. Often, several name servers are specified.

---

# DNS Record Types

* `A` and `AAAA` are address records: mappings from name to IP address.
* `PTR` is a reverse mapping from IP address to name
* `NS` is a pointer to a name server.
* Other record types: `SOA`, `CNAME`, `MX`, `TXT`.

^ DNS has several different types of records, each of which performs a different function. `A` records (and their corresponding IPv6 counterpart `AAAA` records) map from a host name to an IP address. That’s what we’re looking for here, though we’ll bump into some other record types along the way. `CNAME` records are a mapping from an alias to a canonical name. The `SOA` provides some information about a domain, including which name server is its primary authority, and some information to control how records in that domain can be cached.

---

# Send the DNS request

New is Apple iOS 9 & El Capitan

* Send out an `AAAA` request; and
* Send out an `A` request, in parallel.

^ A shiny new feature in Apple’s iOS 9 and Mac OS X El Capitan that I spotted a couple of weeks ago. It will send out DNS requests for an IPv6 `AAAA` record and an IPv4 `A` record, both at the same time. The process is the same, and whichever responds first (more or less) wins.

---

#  Recursive DNS request

Is the record in the name server’s cache?

* **Yep** Is it still valid? (TTL) If so, return the record. Job done.
* **Nope** OK, we’ll need to look it up.

^ Most recursive name servers keep a cache of records they’ve recently looked up. DNS is designed so that records are cached with well defined semantics. Each resource record has a Time To Live (in seconds) associated with it. If the record’s time to live is greater than 0, it’s considered to still be valid and is returned as an answer. Depending on the situation — essentially, the stability of the record in question — a TTL can be anything from a few minutes to a few weeks.

---

# Upstream DNS server

Is our local DNS server configured to have one or more upstream servers?

* **Yep** OK, let’s pass the request off to an upstream and let it figure out the answer.
* **Nope** Damn. We’re going to have to do the hard work ourselves.

^ DNS servers are most often configured in a hierarchy. Chances are you’ll have a DNS server somewhere on your local network. At home, your DSL router will probably act as a recursive DNS server. It will be configured to send DNS requests that it can’t answer up to your ISP’s customer-facing DNS resolver. Your ISP’s DNS servers may well themselves be configured with upstream servers in a hierarchy of their own. Eventually, we get to the top of the hierarchy, with the root DNS servers.

--- 

# Root DNS Servers

* 13 well-known IP addresses of root servers.
* Really, they’re hundreds of machines distributed globally.
* Authoritative for the root zone.

^ The IP addresses of the root domain name servers are well-known, and are distributed by default with most domain name server implementations. We need to know their IP addresses in order to bootstrap the domain name system — it’s not possible to resolve a name without knowing the IP address of a DNS server that can answer the question. While they consist of only 13 IP addresses, really there are hundreds (500+ last time I looked) of machines around the globe. A mechanism called Global Server Load Balancing means that disparate systems can share the same IP address, and when you attempt to make a connection to that IP address, you’ll be routed to a machine that’s geographically (more or less) close to you.

---

# DNS Authority

Root servers are authoritative for the root zone.

Know the canonical answer for who serves each TLD: “.com”, “.net”, “.uk”, etc.

^ The root servers are authoritative for the root zone. This means that they know the canonical answer for the list of name servers (and, through glue records, the IP addresses of those name servers) who are authoritative each top level domain, including country-specific top level domains.

---

# What’s the A record for “google.com”?

^ So we’ve got to the stage where the browser has determined it doesn’t have the IP address for google.com in its own cache. The operating system resolver doesn’t have the IP address either, so we’ve moved on to doing a DNS lookup. Our local caching name server (our DSL router) doesn’t have the answer, so it has asked the ISP’s DNS server. It doesn’t know the answer, and doesn’t have another upstream to punt the request to, so it’s going to figure out the answer for itself (and cache the result).

---

# Root servers

What’s the A record for “google.com”?

* No idea, but here’s the list of name servers for “.com”.
* Oh, and have the IP addresses of those name servers, too.

^ The root name servers cannot hold DNS information for every host on the Internet. That would be crazy. So when you ask the root servers for a specific record, they have no idea what the answer is, but they can help point you in the right direction. So instead of a direct answer to the question, they’ll say, “I dunno” but they’ll help out by giving you a list of hostnames for name servers that serve the “.com” domain — because they do authoritatively know the answer to that. If there’s enough space in the response, they’ll also hand out IP addresses of those name servers, if they happen to know them (and they typically do).

---

# Authoritative servers for “.com”

What’s the A record for “google.com”?

* No idea, but here’s a list of name servers for “google.com”.
* Oh, and have the IP addresses of those name servers, too.

^ The authoritative DNS servers for the “.com” zone don’t even hold all the DNS records for .com. With the sheer number of domains in each TLD zone, and the rate of change of the records in the zone, this would be infeasible. However, thanks to the database of domain names maintained by each TLD registry — extracted from the whois database — the zone does have details of the authoritative name servers for each domain name within the zone. So this time, instead of a direct answer to the question, we get back a list of name servers — and, if possible, their IP addresses — for the “google.com” domain.

---

# Authoritative servers for “google.com”

What’s the A record for “google.com”?

* Hey, I know this! Here’s a list of IP addresses!

^ At last, we’ve got an answer! The authoritative servers for google.com know the list of IP addresses that we should talk to. It’ll return one or more addresses. If there’s more than one, it’ll return the list in a random order — this provides a form of load balancing, in that hopefully each client will split their requests fairly evenly across all the advertised IP addresses.

---

# Figuring out the TCP port

What TCP port should we connect to?

* Figure out from the URL scheme
* Ask the operating system: `getservbyname()`
* Name Service Switch
* `grep '^https.*tcp' /etc/services`

^ As well as providing an abstraction to specify how we resolve hostnames, the Name Service Switch provides other databases, too. One of these is a mapping from well known service names to their port numbers. We ask for the TCP port number for the https service. The answer to this question can typically be found in the file `/etc/services` on most Unix systems. We can see the answer to the question ourselves with a bit of `grep` tomfoolery.

---

# Making a TCP connection

We know the IP address and port. Now we can connect!

^ The browser now initiates a TCP connection, using the BSD socket library. This is a standard Unix abstraction for userspace applications to communicate with each other. The abstraction is such that applications on the same machine can communicate with each other directly over a socket, and applications on different hosts can communicate with each other over TCP/IP. The socket library provides a `connect()` method that allows us to initiate the TCP connection to the IP address for google.com on TCP port 443 (the well known port for https).

---

# TCP: Three way handshake

Open connection, and agree initial sequence numbers.

* -\> SYN
* \<- SYN+ACK
* -\> ACK

^ TCP starts up a connection with a three-way handshake. What we’re doing here is opening each side of the full duplex stream. The browser constructs a TCP packet with the appropriate source and destination IP addresses, the source and destination ports (the source port is chosen arbitrarily), the SYN flag, and a sequence number, which is chosen at random. It then sends this TCP packet out on the appropriate network interface and, through the magic of IP routing, it arrives at the server’s network interface and gets passed up the network stack again.

^ The server responds with a packet directed back to the client’s IP address, with the destination port specified as the source that originated the packet. It sets the ACK flag to acknowledge that it has received the sequence number from the client. It also sets the SYN flag and chooses a random sequence number for keeping track of the stream from the server back to the client.

^ The client (hopefully) gets the packet from the server, notes the sequence number the server has acknowledged, and responds with an ACK to acknowledge receipt of the sequence number for the server -\> client side of the connection.

^ With the three way handshake complete, we have data streams open in both directions, so the server and the client are able to communicate.

---

# Transmission Control Protocol (TCP)

* Ordered data transfer
* Reliable data transfer
* Flow control
* Congestion control

^ Now that we’ve got a data stream open in both directions, we can start to send some data back and forth. The underlying IP protocol is not designed for reliability by itself. The basic premise is that, if there’s a problem with a packet, just drop it on the floor. If a router can’t figure out the next destination for a packet? Drop it. If a link between two routers is congested? Drop some packets to make space. If the server falls off the Internet? Drop its packets.

^ IP doesn’t guarantee the order in which packets emerge from the network either. Since routing decisions are made for each individual packet, some packets might take a different route to their destination. Some might be held up in queues. Some might be dropped and need to be retransmitted.

^ TCP provides extra functionality on top of IP to give us ordered, reliable data transfer. Mostly it achieves this through assigning sequence numbers to the individual packets being sent, and through acknowledging receipt of the packets.

^ We start out slow, with a small congestion window. This means that only a small number of unacknowledged packets can be sent out on the wire at once. When packets are received by the other end, they are acknowledged so the sender knows they have been received. When we’re confident that there is sufficient bandwidth between the sender and receiver to cope with the rate of data being sent, we can increase the congestion window, allowing more packets to be in flight at once. This increases the rate of data being sent. Eventually, we’ll hit a limit where packets start to get dropped. At that point, we’ll scale back the congestion window slightly, having figured out the fastest rate we can reliably transmit data.

---

# Transport Layer Security

Create a secure connection between the client and server.

* Authentication of the server (and, optionally, the client).
* Negotiate a session key.
* Encrypt data between client and server.

^ Transport Layer Security allows us to create a secure connection between the client and the server. This means that communication between the two endpoints is authenticated — the client can verify that a server really is the entity we’re meaning to communicate with (and vice versa, if need be) — and encrypted so that no eavesdropper between the client and server can intercept the communicated data.

^ First of all, the client initiates the secure connection by presenting a list of ciphers — encryption algorithms — and hash functions it supports. The server chooses a particular cipher suite and informs the client of this decision. The server also sends the client its credentials as a digital certificate. The certificate contains the server’s name (which matches the DNS name “google.com”), the certificate authority which has verified the server’s identity, and the public key it proposes we use for negotiating the secure connection.

^ The client has to verify that the server’s credentials are valid. First of all, it checks that the server’s certificate has the same common name as the DNS name we’d intended to communicate with. In this case, both are “google.com” so we’re happy that the certificate corresponds to the DNS name. Now we need to check that the public key is in fact a valid, trustable key. Web browsers are distributed with a well known set of root certificate authorities — entities who are trusted to issue SSL certificates. These certificate authorities undertake to verify that they are issuing certificates to the correct people — usually using some out of band process for verifying identities. For individuals, this might be verifying government-issued photo id. For corporations, this will involve some other verification. The server’s public key will be signed by a certificate authority. We can verify that the certificate is valid and hasn’t been tampered with by checking the signature using the certificate authority’s certificate that’s distributed with the web browser.

^ The client can also check in with the certificate authority to verify that the certificate hasn’t been revoked since it was issued.

^ The client then encrypts a message to the server, with a random number, using the server’s public key. Only the server should be able to decrypt this message, using its corresponding private key. Now both the client and the server have a shared secret they can use to encrypt the remainder of the communication. We have a secure tunnel!

---

# HTTP: GET

`GET / HTTP/1.1`
`Host: google.com`

^ Finally, we can make the HTTP request itself. The client requests the the path we’re looking for (extracted from the URL earlier on).

---

# FIN

`FIN` -\> `ACK` -\> `FIN` -\> `ACK`

TCP/IP Illustrated by W. Richard Stevens

\<https://woss.name/\>

^ That’s all we’ve got time for! There’s still tonnes we could cover. We haven’t really talked about UDP, or IP (version 4 or 6), or the data link layer (Ethernet or Wifi). We could talk about address resolution at the data link layer, routing, fragmentation & reassembly, or many other topics! Hopefully I’ve managed to reveal some of the workings from behind the curtain, and I really hope I’ve enthused you to learn some more. If you are looking to dig into this, the canonical reference is TCP/IP Illustrated by W. Richard Stevens. If you can find it, the first edition might be a little dated, but its much more fun to read — the second edition gets a little bogged down in the details and has less flair.

^ And, of course, you can check out my blog, where I have been meaning to write more about this sort of stuff.

^ Thank you for listening, and enjoy the rest of OSCON!

[1]:	mailto:mathie@woss.name
[2]:	https://twitter.com/mathie