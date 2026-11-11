# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **2614**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop
I created the tunnel loop by modifying the entry for tunnel exit ID 3 in the MyIngress.myTunnel_exact table on switch s2. I changed its output port from port 3 to port 1. Since s1 already forwards packets with tunnel destination ID 3 to s2, changing s2's output port to port 1 caused packets to travel back to s1. Created a continuous forwarding loop between these two switches.

I used sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1" command to modify the table entry, and sent tunnelled packet using h1 python3 send.py 10.14.0.3 "loop" --tunnel 3. After that, I monitored the number of packets processed by both switches using sh grep -c "Processing packet received" logs/s1.txt logs/s2.txt and sh sleep 5; grep -c "Processing packet received" logs/s1.txt logs/s2.txt

Initially, the packet counts were 396(s1) and 394(s2). After five seconds, the counts increased to 1269(s1) and 1268(s2). This continuously increasing packet counts confirm that the packets were repeatedly circulating between the two switches instead of reaching to their intended destination.

IPv4 TTL does not stop the tunnel loop because tunnelled packets use myTunnel_forward instead of ipv4_forward. Although ipv4_forward decrements the TTL, myTunnel_forward does not decrement the inner IPv4 TTL. When IPv4 TTL reaches 0, the packet should be discarded and not forwarded further. However, the tunnel forwarding path does not check the TTL, so it cannot rely on IPv4 TTL to stop the tunnel loop.

## 2. Design: the hop limit
I designed an 8-bit unsigned hop_limit field (1 byte) because it supports values from 0 to 255. This is sufficient for limiting the number of switches a packet can cross without adding much header overhead. Placing it after dst_id in the tunnel header keeps the existing proto_id and dst_id fields in their original positions.

Since we only have two switches, a starting value of 8 gives enough allowance for normal forwarding while ensuring a looping packet is eventually dropped.

The hop limit will be checked and decremented in ingress pipeline before applying the tunnel forwarding table. If the value is zero, the packet is dropped, else it is decremented by one and the packet is forwarded.

The tunnel header definition, parser, and deparser must be updated to recognize and preserve the new field. The sender (send.py) must include the initial hop limit when constructing a tunnelled packet, while the receiver (receive.py) must decode and display the remaining hop limit. This ensures that the hosts and switches use the same tunnel header format.

## 3. Implementation
I updated tunnel.p4 by adding an 8-bit hop_limit field to the tunnel header. In the ingress pipeline, the hop limit is checked before tunnel forwarding. If the value is 0, the packet is dropped. Otherwise, it is decremented by one before the forwarding table is applied.

I also modified send.py to include the hop_limit in the tunnel header. The default value is 8. In receive.py, I updated the tunnel header definition and modified the receiver output to display the remaining hop limit when tunnelled packet arrives.

Both host scripts use the same tunnel header format as the P4 switches. This ensures that the additional field is correctly transmitted and interpreted. I tested delivery to h2 and h3 using an initial hop limit of 8 and both receivers reported a remaining hop limit of 6. This confirms that each of two switches decremented the value once. I also tested normal IPv4 forwarding using ping to confirm that the hop-limit implementation does not affect the IPv4 forwarding.

## 4. Evidence: it works
In delivery.txt file:
---------- Tunnelled packet to h2 ----------
h1 python3 send.py 10.14.0.2 "hop test" --tunnel 2 --hops 8

Output from logs/h2.txt:
Listening on h2-eth0...
Received (tunnelled, dst_id=2, hop_limit=6) from 10.14.0.1 to 10.14.0.2, TTL 64: 'hop test'

---------- Tunnelled packet to h3 ----------
h1 python3 send.py 10.14.0.3 "hop test h3" --tunnel 3 --hops 8

Output from logs/h3.txt:
Listening on h3-eth0...
Received (tunnelled, dst_id=3, hop_limit=6) from 10.14.0.1 to 10.14.0.3, TTL 64: 'hop test h3'

---------- IPv4 Connectivity ----------
h1 ping -c 2 h3

Output:
2 packets transmitted, 2 received, 0% packet loss
64 bytes from 10.14.0.3: icmp_seq=1 ttl=62
64 bytes from 10.14.0.3: icmp_seq=2 ttl=62

In loop-after.txt file:
------- Create forwarding loop -------
sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"

Output:
Done
Modifying entry 2 for exact match table MyIngress.myTunnel_exact

------- Send tunnelled packet with hop limit 8 -------
h1 python3 send.py 10.14.0.3 "loop after" --tunnel 3 --hops 8

Packet was sent with destination id:3 , initial hop limit:8, inner IPv4 TTL: 64

------- Switch processing counts -------
1. First check
logs/s1.txt:229
logs/s2.txt:225

2. Second check
logs/s1.txt:229
logs/s2.txt:225

------- Analysis -------
The packet was successfully forwarded 7 times. On its 8th switch processing event, the hop limit was decremented to 0 and the packet was dropped. This shows that it prevents an endless forwarding loop.

## 5. Evaluation
I predict that the packet will not be delivered to h2. The new program on s1 uses a 5-byte tunnel header containing proto_id, dst_id, and hop_limit. While the old program on s2 expects the original 4-byte header containing only proto_id and dst_id. Therefore, s2 may interpret the new hop_limit byte as the beginning of the inner IPv4 header. This shifts the expected packet fields and may cause packet loss. I also predict that the old s2 program will not enforce the new hop limit, so the loop-protection behaviour may differ from the upgraded network.

a) The mixed rollout (mixed.txt):
------- Mixed-version configuration -------
Started the topology using the new program(s1) and original program(s2):
python3 topo.py --id 2614 --s2-program tunnel_v1.p4

------- Prediction -------
The old s2 program expects a 4-byte tunner header, while the new s1 program sends a 5-byte header containing an additional 8-bit hop_limit field. My prediction is that s2 would misinterpret the packet as it does not recognise the additional byte.

------- Tunnel packet delivery test -------
h1 python3 send.py 10.14.0.3 "mixed retry" --tunnel 2

The old s2 program successfully matched destination ID 0002 and transmitted the packet through port 2.
However, the capture on h2 showed that the packet was no correctly aligned for IPv4 parsing.

Captured bytes:
1212 0800 0002 0745 0000 2700 ...

The old s2 parser expected the IPv4 header immediately after the 4-byte tunnel header. Instead, it treated the hop-limit byte as the beginning of the IPv4 header. The packet reached h2 but its header was misinterpreted.

------- Mixed-version loop test -------
Before sending:
logs/s1.txt:231
logs/s2.txt:227

Sent:
h1 python3 send.py 10.14.0.3 "mixed loop" --tunnel 3 --hops 8

After sending:
logs/s1.txt:240
logs/s2.txt:235

After another 5 seconds:
logs/s1.txt:240
logs/s2.txt:235

The counts stopped increasing. This confirms that the loop does not continue.
The new s1 program enforced the hop limit, while the old s2 program did not. The packet still returned to s1, where the hop limit was decremented until the packet was dropped

------- Conclusion -------
The mixed-version test showed that adding a field to a packet header without ensuring backward compatibility can cause parsing errors. Although the hop limit still stopped the loop, the header change was not fully compatible with the old switch program.

b) Safe rollout strategy
The first option is to introduce a new EtherType for the updated tunnel format. New switches can recognise the new EtherType and parse the additional hop-limit field, while the old switches will not mistake the new packet format for the old one. However, old switches may drop these packets, affecting connectivity until they are upgraded.

Second option is to implement backward-compatible parsing before introducing the new header. Switches can first be upgraded to recognise both the old and new header formats. Once all switches support both formats, the network can begin using the new format. It requires additional parser logic but allows a more controlled transition.

c) Data plane versus control plane loop protection
The hop limit provides data plane protection by limiting how many times an individual packet can be forwarded. It can stop a packet from circulating indefinitely even if a forwarding loop is introduced by incorrect forwarding rule.

While check_paths() provides control plane protection by checking forwarding paths for problems before they are used. It can identify invalid or looping paths in the planned network configuration. However, it cannot guarantee that every packet will avoid a loop caused by later changes or incorrect switch rules.

Both are used in real networks because hop limit provides runtime safeguard for packets that encounter a loop, while check_paths() prevents problematic paths from being configured.

d) Hardware switch cost
The hop-limit mechanism adds an extra 8-bit field to the tunnel header, increasing the header size by one byte per packet. Each switch must perform a comparison, decrement the value, and drop the packet when limit is 0.

These operations require additional parser, metadata, and processing resources in the switch. On a hardware switch, the implementation must fit within the available pipeline stages and processing capabilities to maintain line-rate forwarding. If the hardware supports these simple operations efficiently, the additional cost should be relatively small, although the exact impact depends on the switch architecture.

## AI use
I only used ChatGPT throughout this assignment for troubleshooting and explaining the P4 implementation and test results. This AI assistance tool helped me understand the BMv2 logs, interpret packet captures, implement and test the tunnel hop-limit mechanism, and understand the concepts discussed in Section 5 (Evaluation). For examples, mixed-version rollout and backward compatibility. I used the explanation to improve my understanding and verify my implementation through testing.

## Evidence files
| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
