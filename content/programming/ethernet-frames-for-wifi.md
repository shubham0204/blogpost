---
title: Why Do We Get Ethernet II Frames Over Wi-Fi On MacOS?
tags:
  - operating-systems
---
I was exploring network packets with [`termshark`](https://github.com/gcla/termshark) on macOS 15.6 (Apple M4), prompting Claude to provide me some interesting queries that can I apply on the packet capture stream to learn some fascinating concepts. On filtering the packet stream with the query `dns.qry.name contains ".com"`, I was able to filter DNS packets with queries for domain names containing `.com` in them.

`termshark` provides a drill-down view of each packet, revealing protocols used at each layer of TCP/IP or OSI model during the packet transfer. The following protocols are used:

1. [DNS (Domain Name System)](https://en.wikipedia.org/wiki/Domain_Name_System) at application-layer (layer 7) defines queries, answers, and their properties
2. [UDP (User Datagram Protocol)](https://en.wikipedia.org/wiki/User_Datagram_Protocol) at transport-layer (layer 4) defines parameters for end-to-end delivery of the payload
3. [IP (Internet Protocol)](https://en.wikipedia.org/wiki/Internet_Protocol) at networking-layer (layer 3) defines addressing and routing information
4. Ethernet at the data-link and physical-layer (layer 1 and 2) is the closest to the physical medium 

![[Pasted image 20250907201938.png]]

As I use Wi-Fi (WLAN) to connect my Mac to the router for an internet connection, I was expecting an [IEEE 802.11](https://en.wikipedia.org/wiki/IEEE_802.11) frame in the view above. [Ethernet](https://en.wikipedia.org/wiki/Ethernet) is for wired connections and I was not expecting it in the DNS packet drill-down. 

As it turns out, the Wi-Fi network driver *does* use the IEEE 802.11 protocol to transfer bits across the medium i.e. from my Mac to the router wirelessly. When the network driver passes the data to the operating-system, it converts them to Ethernet frames. Wireshark or `termshark` operate at the same abstraction level as that of the OS, and hence they view only Ethernet frames even if the underlying data was transferred with IEEE 802.11. 

As Wi-Fi (WLAN, wireless LAN) was introduced after Ethernet, the network driver for Wi-Fi had to perform this frame conversion to avoid modifications in the operating system code necessary to understand IEEE 802.11 frames. This also made the network driver compatible with many other operating systems as there was no need to write additional code for parsing IEEE 802.11 payloads.

### References

1. https://superuser.com/a/1242481
2. https://networkengineering.stackexchange.com/questions/48496/why-are-wlan-packages-translated-to-ethernet-packages-by-wireshark