# DNS Tunnel - Experimental File Transfer

This project is a minimal educational implementation of a DNS tunnel for
transferring a file from a server to a client. The server listens for
UDP/DNS packets built with [Scapy](https://scapy.net/), while the client
requests successive file chunks through DNS `TXT` queries.

> **Safety and legal notice**
>
> DNS tunneling can bypass network controls and is used in real attacks to
> exfiltrate data. Use this project only in an isolated laboratory or on
> infrastructure for which you have explicit permission. Do not use it on
> public networks, university/company infrastructure, or data that does not
> belong to you.

## Exercise objectives

This assignment provides practice with:

1. the principles of DNS tunneling and query/response abuse;
2. configuring `NS` and `A` records in a laboratory domain;
3. manually constructing IP/UDP/DNS packets;
4. splitting a file into chunks, Base64-encoding them, and sending them in
   DNS `TXT` responses;
5. handling UDP packet loss by retransmitting requests;
6. verifying file integrity with an MD5 checksum.

Useful background material:

- [DNS Tunnel](https://dnstunnel.de/);
- [DNS tunneling detection and mitigation](https://www.prosec-networks.com/en/blog/dns-tunneling-erkennen/);
- [DNS tunneling - Catchpoint](https://www.catchpoint.com/network-admin-guide/dns-tunneling);
- [dnstt](https://www.bamsoftware.com/software/dnstt);
- [iodine](https://github.com/yarrick/iodine).

## Project files

| File | Purpose |
|---|---|
| [`server.py`](./server.py) | Sniffs DNS queries on UDP port `5353` and responds with `TXT` chunks from `secret.txt`. |
| [`client.py`](./client.py) | Requests chunks in order, retries after a timeout, and writes the result to `downloaded.txt`. |
| [`secret.txt`](./secret.txt) | Test file stored on the server. |
| [`downloaded.txt`](./downloaded.txt) | File produced on the client after the transfer. |

## How it works

### Client request

For chunk `n`, the client constructs the following name:

```text
req-n.secret.txt.soimiipython.dev.
```

In [`client.py`](./client.py), this name is generated with:

```python
qname = f"req-{chunk_index}.{FILE_NAME}.{DOMAIN}"
```

The query is a DNS `TXT` request sent to `SERVER_IP` on UDP port `5353`.
`chunk_index` starts at `0` and is incremented only after a valid response
has been received and decoded.

### Server response

In [`server.py`](./server.py), the server:

1. checks that the packet contains `DNS` and `DNSQR` layers;
2. checks that the requested name contains the configured domain;
3. extracts the chunk number from the `req-` prefix;
4. opens `secret.txt` in binary mode;
5. seeks to `chunk_index * CHUNK_SIZE` and reads up to `100` bytes;
6. Base64-encodes the chunk;
7. sends the encoded data in a `DNSRR(type="TXT")` response.

After the last chunk, the server sends the literal `EOF` marker. The client
stops when it receives this marker.

The exchange is:

```text
client -- UDP/DNS TXT: req-0... --> server
client <-- UDP/DNS TXT: Base64(chunk 0) -- server
client -- UDP/DNS TXT: req-1... --> server
client <-- UDP/DNS TXT: Base64(chunk 1) -- server
...
client <-- UDP/DNS TXT: EOF -------- server
```

### Transfer reliability

UDP does not guarantee delivery, ordering, or uniqueness. The client uses a
simple **stop-and-wait** mechanism:

- it waits up to `2` seconds for the current chunk;
- after a timeout, it sends the same request again;
- it increments the chunk index only after successful decoding.

The server is stateless: it can serve any requested index because it rereads
the file and seeks directly to the calculated offset. The current prototype
does not persist progress and does not automatically resume after the client
or network connection is restarted. A new client run starts at chunk `0`.
For a complete implementation of the assignment, add a persistent checkpoint
file, index validation, and resumption from the last confirmed chunk.

## Tools and technologies

- **Python 3** - implementation language;
- **Scapy** (`scapy.all`) - builds and decodes `IP`, `UDP`, `DNS`, `DNSQR`,
  and `DNSRR` layers, sends queries, and captures packets;
- **Base64** (`base64`) - converts bytes into a representation suitable for
  transport in a `TXT` field;
- **hashlib.md5** - calculates the checksum of the received file;
- **Packet capture/raw packet privileges** - sniffing and sending packets may
  require administrator privileges;
- **dig** - recommended for independently testing DNS `NS` and `A` records.

Base64 provides neither encryption nor confidentiality. Anyone capturing the
traffic can decode the chunks.

## Installation

A virtual environment is recommended:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install scapy
```

Verify the installation:

```bash
python -c "from scapy.all import IP, UDP, DNS, DNSQR; print('Scapy OK')"
```

## Configuration

Before running the programs, update the values in both scripts to match the
laboratory environment.

### `server.py`

```python
DOMAIN = "soimiipython.dev"
FILE_PATH = "secret.txt"
CHUNK_SIZE = 100
```

- `DOMAIN` is the laboratory domain used in queries;
- `FILE_PATH` is the file made available by the server;
- `CHUNK_SIZE` is the maximum number of source bytes per chunk before
  Base64 encoding.

### `client.py`

```python
SERVER_IP = "34.16.244.55"
DOMAIN = "soimiipython.dev"
FILE_NAME = "secret.txt"
OUTPUT_FILE = "downloaded.txt"
```

- `SERVER_IP` is the private or public laboratory server address;
- `DOMAIN` must match the server's domain;
- `FILE_NAME` must match the file component used in the query;
- `OUTPUT_FILE` is created or overwritten by the client at the end.

Do not keep real addresses, credentials, or sensitive data in a public
repository.

## DNS configuration for a laboratory

The current project sends packets directly to an IP address and port `5353`.
It does not run a conventional DNS daemon on port `53` and does not
automatically create a DNS zone.

If the assignment requires delegation of a subdomain, configure the `NS` and
`A` records with your DNS provider according to the laboratory instructions.
Test resolution first:

```bash
dig NS soimiipython.dev
dig A soimiipython.dev
dig @<nameserver> A <subdomain>.soimiipython.dev
```

In a local laboratory, a private network, test resolver, or port `5353` is
safer than modifying public DNS. Public DNS records are not required for a
direct client/server test when `SERVER_IP` is reachable and the packet can
reach the selected port.

## Running the project

### 1. Start the server

On the machine containing `secret.txt`:

```bash
sudo python3 server.py
```

Expected output:

```text
dns tunnel asculta pe 5353
```

The server uses `sniff(filter="udp port 5353", ...)`, so the laboratory
firewall must allow UDP traffic on this port.

### 2. Start the client

On the client, after configuring `SERVER_IP`:

```bash
sudo python3 client.py
```

The client prints messages such as `bucata 0` for each chunk. If a packet is
lost, it prints `timeout pentru bucata 0. retrimitem`. At the end, it prints
the output path and calculated MD5 checksum.

## Verifying the result

Calculate the checksum of the original and received files, ideally on their
respective machines:

```bash
md5sum secret.txt
md5sum downloaded.txt
cmp --silent secret.txt downloaded.txt && echo "Files are identical"
```

The MD5 values should match for this exercise. MD5 is used here to
demonstrate integrity, not cryptographic security. For real applications,
prefer SHA-256:

```bash
sha256sum secret.txt downloaded.txt
```

## Known limitations

- the server accepts queries when `DOMAIN` appears anywhere in the name
  instead of strictly validating the DNS suffix;
- `chunk_index` is not validated before conversion to `int`;
- there is no authentication, encryption, access control, or protection
  against modified responses;
- there are no sequence numbers in the payload, explicit acknowledgements,
  sliding window, or deduplication;
- the client keeps the complete file in memory and writes it only at the end;
- reconnecting after the process stops does not resume from a checkpoint;
- `EOF` is a special application marker, not a standard DNS message;
- the implementation depends on Scapy raw packets and capture filters, so
  behavior may differ between Linux, macOS, and Windows;
- port `5353` is used as a laboratory port, not as a complete DNS resolver.

These limitations are intentional in this minimal educational example. A
more robust project would need a session protocol, transfer identifier,
per-chunk checksums, persistent checkpoints, configurable timeouts, rate
limiting, and strict DNS-name validation.

## Detection and mitigation

Network administrators can investigate DNS tunnels using indicators such as:

- unusually large volumes of queries or `TXT` responses;
- long DNS labels with high entropy and Base32/Base64-like characters;
- many unique subdomains or repetitive numeric sequences;
- regular query frequency and domains with poor reputation;
- external resolvers being used instead of the organization's resolver.

Common defensive measures include:

- allowing DNS only through authorized resolvers;
- monitoring and logging DNS queries;
- applying rate limits and policies for record types;
- blocking malicious domains and resolvers;
- using entropy, length, frequency, and behavior-based detection;
- correlating DNS analysis with firewall, EDR, and endpoint monitoring.
