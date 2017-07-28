# yapcap: Yet Another PCAP module for Python

This module allows you to easily read pcap and PcapNG files in Python.

I was looking around for a pcap library to use to read at some DNS
captures, and struggled to find one that was well-maintained, supports
Python 3, supports both pcap and pcapng formats, and had an intuitive
API. Eventually I decided to roll my own.

## General Usage

Usage is something like this:

```python
with open('myfile.pcap', 'rb') as pcapfp:
    for packet in yapcap.packets(pcapfp):
        # access metadata as packet.timestamp, packet.info.pkt_type, ...
        # access contents as packet.data
```

If you have compressed packet captures, you may want to do use
something like the `gzip` or `bz2` modules to read them:

```python
with gzip.open('myfile.pcap.gz') as pcapfp:
    for packet in yapcap.packets(pcapfp):
        ...
```

## Exceptions

If an error is found in the input file, a yapcapError will be raised.
pcap and PcapNG files have their own classes:

```
+-- yapcapError
    |
    +-- PcapError
    |
    +---PcapNGError
```

Each exception raised includes an error message that only appears in a
single place in the code, in case you want to look at the code which
raised the error and try to figure out why you got it.

## Metadata

The following metadata is present for all packets:

| Name        | Description |
|-------------|-------------|
| `cap_type`  | Either 'pcap' or 'pcapng' |
| `version`   | Version of pcap ("2.4") or PcapNG ("1.0") |
| `snaplen`   | Maximum length of a captured packet |
| `linktype`  | The type of link-layer header metadata for the packet |
| `timestamp` | The epoch value where the packet arrived, as a `decimal.Decimal` value |
| `origlen`   | The original length of the packet |
| `pkttype`   | The type of the packet: IPv4, IPv6, OSI, or IPX |

We use a `decimal.Decimal` type for the timestamp because the
precision of the timestamps in the file may be too much to fit in a
`float` type or `datetime.datetime` object (both only support
microsecond accuracy).

No metadata describes the length of the captured packet. Use
`len(packet.data)` if you need this. If a packet was truncated, the
`orig_len` value is longer than the length of the captured packet.

## Link-layer Header Metadata

Depending on the format used, there may be additional link-level
header information.

The yapcap package currently supports:

* LINKTYPE_NULL
* LINKTYPE_ETHERNET
* LINKTYPE_RAW
* LINKTYPE_LOOP
* LINKTYPE_IPV4
* LINKTYPE_IPV6

If the `link_type` is `LINKTYPE_ETHERNET`, then the packet metadata
will also have the source and destination MAC address of the packet:

| Name        | Description |
|-------------|-------------|
| `src_mac`   | The source MAC address of the packet, as a string like `"70:85:c2:3b:e8:f0"` |
| `dst_mac`   | The destination MAC address of the packet, as a string like `"00:01:2e:78:08:b1"` |

The full list of link-layer header types can be found here:

http://www.tcpdump.org/linktypes.html

For PCAP files, the link-level header is the same for all packets in
the capture. For PcapNG, each packet may have different link-level
header types.

## pcap Files

pcap files can be read directly using the PcapFile class. You will
probably want to decode the captured data using the linklayer.Decoder
class. At end of file an EOFError exception is raised.

```python
with open('blessedpackets.pcap', 'rb') as fp:
    pcapfile = yapcap.PcapFile(fp)
    decoder = yapcap.linklayer.Decoder(pcapfile.network,
                                       pcapfile.byte_order)
    try:
        while True:
            pkt_hdr, cap_data = pcapfile.read_pkt()
            pkt_data, pkt_type, pkt_info = decoder.decode(cap_data)
            # do something with pkt_hdr or pkt_data
    except EOFError"
        pass
```

## PcapNG Files

The PcapNG format provides a lot of additional information beyond what
pcap does. It is structured in blocks, which may provide packet
information, or may provide other information such as starting a new
block, or defining interface characteristics.

The following additional information may be present for a packet in a
PcapNG file:

| Name        | Description |
|-------------|-------------|
| `shb_comment`  | Comment from the section header block |
| `shb_hardware` | Hardware of the machine that wrote the capture |
| `shb_os`       | OS of the machine that wrote the capture, like "Linux 4.11.0" |
| `shb_userappl` | Application that wrote the capture, like "Dumpcap 1.12.1" |
| `if_comment`   | Comment from the interface block |
| `if_name`      | Name of the interface, like "eth0" |

TODO: finish the descriptions, document how to read blocks directly...

## Performance Considerations

No attention has been given to performance, with the primary goal
being to build a robust, Pythonic module.

If things are running too slowly, you can use [PyPy](http://pypy.org/)
for your programs, which will probably result in much faster execution
times.
