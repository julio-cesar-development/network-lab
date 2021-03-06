# Symmetric VXLAN routing with BGP EVPN

This is similar to `lab-juniper-vqfx-vxlan-asymmetric` but it uses an
L3 EVPN service instead. Type-5 routes are advertised and there is no
need to declare the IRB on all the VTEP:

    juniper@QFX1> show route receive-protocol bgp 172.29.1.2
    
    inet.0: 8 destinations, 9 routes (8 active, 1 holddown, 0 hidden)
    
    VRF-OVERLAY.inet.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
    
    :vxlan.inet.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
    
    inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
    
    VRF-OVERLAY.inet6.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
    
    bgp.evpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
      Prefix                  Nexthop              MED     Lclpref    AS path
      4:172.29.1.2:0::010400000000000019:172.29.1.2/296 ES
    *                         172.29.1.2                   100        I
      5:172.29.1.2:1::0::172.27.2.0::24/248
    *                         172.29.1.2                   100        I
    
    VRF-OVERLAY.evpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
      Prefix                  Nexthop              MED     Lclpref    AS path
      5:172.29.1.2:1::0::172.27.2.0::24/248
    *                         172.29.1.2                   100        I
    
    default-switch.evpn.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
    
    __default_evpn__.evpn.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
      Prefix                  Nexthop              MED     Lclpref    AS path
      4:172.29.1.2:0::010400000000000019:172.29.1.2/296 ES
    *                         172.29.1.2                   100        I

We can see the route is correctly installed into `VRF-OVERLAY` VRF:

juniper@QFX1> show route table VRF-OVERLAY 172.27.2.10 extensive

    VRF-OVERLAY.inet.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
    172.27.2.0/24 (1 entry, 1 announced)
    TSI:
    KRT in-kernel 172.27.2.0/24 -> {composite(1765)}
            *EVPN   Preference: 170/-101
                    Next hop type: Indirect, Next hop index: 0
                    Address: 0xdd02270
                    Next-hop reference count: 2
                    Next hop type: Router, Next hop index: 1761
                    Next hop: 172.29.1.2 via xe-0/0/1.0, selected
                    Session Id: 0x0
                    Protocol next hop: 172.29.1.2
                    Composite next hop: 0xc9ae0e0 1765 INH Session ID: 0x0
                      VXLAN tunnel rewrite:
                        MTU: 0, Flags: 0x0
                        Encap table ID: 0, Decap table ID: 4
                        Encap VNI: 9001, Decap VNI: 9001
                        Source VTEP: 172.29.1.1, Destination VTEP: 172.29.1.2
                        SMAC: 02:05:86:71:e1:00, DMAC: 02:05:86:71:be:00
                    Indirect next hop: 0xba77d70 131070 INH Session ID: 0x0
                    State: <Active Int Ext>
                    Age: 7:40       Metric2: 1
                    Validation State: unverified
                    Task: VRF-OVERLAY-EVPN-L3-context
                    Announcement bits (1): 2-KRT
                    AS path: I
                    Composite next hops: 1
                            Protocol next hop: 172.29.1.2 Metric: 1
                            Composite next hop: 0xc9ae0e0 1765 INH Session ID: 0x0
                              VXLAN tunnel rewrite:
                                MTU: 0, Flags: 0x0
                                Encap table ID: 0, Decap table ID: 4
                                Encap VNI: 9001, Decap VNI: 9001
                                Source VTEP: 172.29.1.1, Destination VTEP: 172.29.1.2
                                SMAC: 02:05:86:71:e1:00, DMAC: 02:05:86:71:be:00
                            Indirect next hop: 0xba77d70 131070 INH Session ID: 0x0
                            Indirect path forwarding next hops: 1
                                    Next hop type: Router
                                    Next hop: 172.29.1.2 via xe-0/0/1.0
                                    Session Id: 0x0
                                    172.29.1.2/32 Originating RIB: inet.0
                                      Metric: 1     Node path count: 1
                                      Forwarding nexthops: 1
                                            Nexthop: 172.29.1.2 via xe-0/0/1.0
                                            Session Id: 0

Note the destination VTEP and the encap/decap VNI. The route is also
present in the FIB:

    juniper@QFX1> show route forwarding-table table VRF-OVERLAY destination 172.27.2.10 extensive
    Routing table: VRF-OVERLAY.inet [Index 4]
    Internet:
    Enabled protocols: Bridging, All VLANs,
    
    Destination:  172.27.2.0/24
      Route type: user
      Route reference: 0                   Route interface-index: 0
      Multicast RPF nh index: 0
      P2mpidx: 0
      Flags: sent to PFE
      Nexthop:
      Next-hop type: composite             Index: 1765     Reference: 2
      Next-hop type: indirect              Index: 131070   Reference: 2
      Nexthop: 172.29.1.2
      Next-hop type: unicast               Index: 1761     Reference: 4
      Next-hop interface: xe-0/0/1.0

We can ask the FPC (`start shell pfe network fpc0`) for more details
on the composite route (but mostly, we also see the details in the
previous `show route` command):

    FXPC0(QFX1 vty)# show nhdb id 1765 extensive
       ID      Type      Interface    Next Hop Addr    Protocol       Encap     MTU               Flags  PFE internal Flags
    -----  --------  -------------  ---------------  ----------  ------------  ----  ------------------  ------------------
     1765    Compst  -              -                      IPv4             -     0  0x0000000000000000  0x0000000000000000
    
    BFD Session Id: 0
    
    Composite NH:
      Function: Tunnel Function
      Hardware Index: 0x0
      Composite flag: 0x0
      Composite pfe flag: 0xe
      Lower-level NH Ids: 131070
      Derived NH Ids:
      Tunnel Data:
          Type     : VXLAN
          Tunnel ID: 806354944
          Encap VRF: 0
          Decap VRF: 4
          MTU      : 0
          Flags    : 0x0
          AnchorId : 0
          Encap Len: 53
          Encap    : 0x01 0x01 0x01 0x1d 0xac 0x00 0x00 0x00
                     0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
                     0x00 0x02 0x01 0x1d 0xac 0x00 0x00 0x00
                     0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
                     0x00 0x29 0x23 0x00 0x00 0x29 0x23 0x00
                     0x00 0x02 0x05 0x86 0x71 0xbe 0x00 0x02
                     0x05 0x86 0x71 0xe1 0x00
          Data Len : 0
         VPFE: 0 PFE : 0 Port: 0 Flags: 0x0
         Egress_NH: 131070 egr_dsc_install: 1 egr_dsc_install_mask: 0x1
         Agg_member: 0 num_tags: 0 tokens_per_app: 1 n_cnhs = 0
        L2 Rewrite[0]: 00
        MPLS TAGS: num_of_tags = 0
    
    Application type: "DEFAULT"
    ===================================================
    
    for fe_id = 0, Ingress NH handle:
    handle 0x207b9990, flags 0x0, refcount 1
    NH installed at addr NH 833, INT_SEQ 0x10001a09
    Raw dump of the nh words of size 1 words
            0x10000203
    SEQ [10000203]  Interm, SIZE 1, NO_ACT 0, USE_REMAP 0, NEXT_ADDR: 00040, SZ: 3
    Nexthop points to:
    SEQ [10001bf1]  Interm, SIZE 3, NO_ACT 0, USE_REMAP 0, NEXT_ADDR: 0037e, SZ: 1
    Nexthop points to:
    SEQ [10001a01]  Interm, SIZE 1, NO_ACT 0, USE_REMAP 0, NEXT_ADDR: 00340, SZ: 1
    Nexthop points to:
    SEQ [20280001]   Final, SIZE 1, EGPRT_VAL 1, PRT_TYP 1, VPFE 000, GRPID 01,
    
    
    
    
    Egress Next-hop on pfe fe_id 0:
    ------------------------------------
    
    ctr_idx 0x1f7cf,             1370 pkts,           142480 bytes
    
    L2 descriptor
    ==============
    
    des-type    p_next    Shared?   Primary     Last        desc_count  app-type    des-addr
                                    des-size    des-size
    --------    -----     -------   --------    --------    --------    --------    --------
    Private     No        Yes(1    )0           6           1           0           0x31573
    
    Des addrs       : 0x31573
    
    
        F:            TIX  tix: 4
        F:        P_CONST  bv_flags: 2  address: 12
     [  12] Refcount    98 SMAC 02:05:86:71:e1:00
        T:           DMAC  dmac: 000002058671be00
        T: P_TUNNEL_ENCAP  tun_ptr: 61, incL2Len = 20, incL3Len = 20
    
    Tunnel tbl idx:   61 (Refcount:     1)
    
    Hdr_type = TUN_HDR_TYPE_IPV4_UDP
    Buff_seq = TUNNEL_BUFF_SEQ_TUN_TMPLT_MPLS
    Write control Flags:
        TUN_WR_CNTL_ERW_UDP_CONFIG_REGISTER[0]
    Tunnel Encap buffer contents (num_of_bytes = 23):
     buff[00] = 0x85  buff[01] = 0x8e  buff[02] = 0x00  buff[03] = 0x45  buff[04] = 0x00
     buff[05] = 0x00  buff[06] = 0x00  buff[07] = 0x00  buff[08] = 0x00  buff[09] = 0x00
     buff[10] = 0x00  buff[11] = 0x40  buff[12] = 0x11  buff[13] = 0xcd  buff[14] = 0xcf
     buff[15] = 0xac  buff[16] = 0x1d  buff[17] = 0x01  buff[18] = 0x01  buff[19] = 0x00
     buff[20] = 0x00  buff[21] = 0x00  buff[22] = 0x00
    
        T:TUNNEL_IPV4_DIP  tun_ipv4_dest: ac1d0102
        F:         P_NEXT  p_next: 201983
    
    Flabel descr for App "DEFAULT" Flabel_id: 323701:
    ============================================================
    
    des-type    p_next    Shared?   Primary     Last        desc_count  app-type    des-addr
                                    des-size    des-size
    --------    -----     -------   --------    --------    --------    --------    --------
    Public      No        No (0    )3           2           1           0           0x1495
    
    Des addrs       : 0x1495
    
     Flabel : 323701 Segment table  index: : 79[1] Page Table index : 144[6] desc start addr:: 5269[1]
    
        F:        COUNTER  counter: 128975  cix_tc_en: 0
        F:         P_NEXT  p_next: 202099
    
      Routing-table id: 0

We can spot VNI 9001 as 0x23 0x29 bytes in the encap data.

Also see [Type-2 and Type-5 EPVN on vQFX 10k in UnetLab][1].

[1]: https://networkop.co.uk/blog/2016/10/26/qfx-unl/

## QoS

There is also a small experiment with CoS. On Linux, we map the SKB
priority 4 to 802.1p "CA" class (3):

    # ip link set bond0.583 type vlan egress 4:3
    # ip -d l l dev bond0.583
    5: bond0.583@bond0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
      link/ether 50:54:33:00:00:0d brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 0 maxmtu 65535
      vlan protocol 802.1Q id 583 <REORDER_HDR>
        egress-qos-map { 4:3 } addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535

On Linux, this is a bit complex to directly set SKB priority from ToS.
Using `ping -Q 255` will lead to a SKB priority of 4, then translated
to 802.1p 3:

    10:15:24.475048 50:54:33:00:00:0d > 00:00:5e:00:01:01, ethertype 802.1Q (0x8100), length 102: vlan 583, p 3, ethertype IPv4, (tos 0xff,CE, ttl 64, id 50576, offset 0, flags [DF], proto ICMP (1), length 84)
        172.27.1.10 > 172.27.2.10: ICMP echo request, id 404, seq 3, length 64

SKB priority can also be set from Netfilter:

    iptables -t mangle -A POSTROUTING [...] -j CLASSIFY --set-class 0:4

Or directly from C:

    setsockopt(s, SOL_SOCKET, SO_PRIORITY, &prio, sizeof(prio));

On QFX, the classifier on a `family ethernet-switching` interface is
`ieee8021p-default`:

    juniper@QFX1> show class-of-service interface ae0.0
      Logical interface: ae0.0, Index: 551
    Object                  Name                   Type                    Index
    Classifier              ieee8021p-default      ieee8021p                  11
    juniper@QFX1> show class-of-service classifier name ieee8021p-default
    Classifier: ieee8021p-default, Code point type: ieee-802.1, Index: 11
      Code point         Forwarding class                    Loss priority
      000                best-effort                         low
      001                best-effort                         low
      010                best-effort                         low
      011                fcoe                                low
      100                no-loss                             low
      101                best-effort                         low
      110                network-control                     low
      111                network-control                     low
    juniper@QFX1> show class-of-service forwarding-class
    Forwarding class                       ID      Queue  Policing priority  No-Loss   PFC priority
      best-effort                          0         0         normal        Disabled
      fcoe                                 1         3         normal        Enabled
      no-loss                              2         4         normal        Enabled
      network-control                      3         7         normal        Disabled

So, we should get in the queue 3. Unfortunately, it seems statistics
on vQFX are messed up:

    juniper@QFX1> show interfaces queue ae0 forwarding-class fcoe
    Physical interface: ae0, Enabled, Physical link is Up
      Interface index: 640, SNMP ifIndex: 501
    Forwarding classes: 16 supported, 4 in use
    Egress queues: 12 supported, 4 in use
    Queue: 3, Forwarding classes: fcoe
      Queued:
        Packets              :  16045690984503098046                     0 pps
        Bytes                :  17039393963205620466                     0 bps
      Transmitted:
        Packets              :                     0                     0 pps
        Bytes                :                     0                     0 bps
        Tail-dropped packets : Not Available
        RL-dropped packets   :                     0                     0 pps
        RL-dropped bytes     :                     0                     0 bps
        Total-dropped packets:  16045690984503098046                     0 pps
        Total-dropped bytes  :  17039393963205620466                     0 bps

We need to setup a rewrite rule to copy the forwarding class to the
DSCP. We can use this one:

    juniper@QFX1> show class-of-service rewrite-rule name dscp-default
    Rewrite rule: dscp-default, Code point type: dscp, Index: 31
      Forwarding class                    Loss priority       Code point
      best-effort                         low                 000000
      best-effort                         high                000000
      fcoe                                low                 101110
      fcoe                                high                101110
      no-loss                             low                 001010
      no-loss                             high                001100
      network-control                     low                 110000
      network-control                     high                111000

We should be able to use:

    set class-of-service interfaces xe-0/0/1 unit 0 rewrite-rules dscp default

However, it doesn't seem to work on vQFX...
