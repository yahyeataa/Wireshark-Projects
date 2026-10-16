# DNS Traffic Analysis

## Objective

Analyze a DNS A-record lookup and define a compact baseline for successful query/response behavior in the captured lab environment.

## Executive summary

The capture contains one DNS query from `192.168.50.50` to `192.168.0.1` and one successful matching response. The lookup asks for an A record for `us.pool.ntp.org` and completes in **6.946 ms**.

```text
192.168.50.50                                    192.168.0.1
      |  DNS query: A us.pool.ntp.org                  |
      |  UDP 65282 → 65333, ID 0x002b                 |
      |---------------------------------------------->|
      |  DNS response: No error, ID 0x002b             |
      |  UDP 65333 → 65282                             |
      |<----------------------------------------------|
                 6.946 ms request/response time
```

## Packet timeline

| No. | Time (s) | Source → Destination | UDP ports | DNS event | Transaction ID | Length |
|---:|---:|---|---|---|---|---:|
| 1 | 0.000000 | `192.168.50.50` → `192.168.0.1` | 65282 → 65333 | Standard query, A `us.pool.ntp.org` | `0x002b` | 75 B |
| 2 | 0.006946 | `192.168.0.1` → `192.168.50.50` | 65333 → 65282 | Standard query response, No error | `0x002b` | 540 B |

The shared transaction ID and reversed IP/UDP tuple correlate the response to the query.

## Query analysis

| Field | Observed value |
|---|---|
| Requester | `192.168.50.50` |
| Resolver | `192.168.0.1` |
| Query name | `us.pool.ntp.org` |
| Query type | `A` (host address) |
| Query class | `IN` |
| Transaction ID | `0x002b` |
| DNS flags | `0x0100` — standard query with recursion desired (RD) |
| Questions | 1 |
| UDP ports | 65282 → 65333 |

![DNS query details](Screenshots/DNS_Screenshots/04_dns_query_details.png)

## Response analysis

| Field | Observed value |
|---|---|
| Response code | `No error` (`RCODE 0`) |
| DNS flags | `0x8180` — standard query response |
| Questions | 1 |
| Answer RRs | 15 |
| Authority RRs | 6 |
| Additional RRs | 2 |
| Response time | 6.946 ms |
| Payload size | 540 B |

The packet-list summary visibly includes A-record addresses such as `67.129.68.9`, `69.44.57.60`, and `207.234.209.181`. The full response contains multiple records, which is expected for an NTP pool hostname.

![DNS response details](Screenshots/DNS_Screenshots/05_dns_response_details.png)

## DNS statistics baseline

Wireshark's DNS statistics window reports the following for the two-packet sample:

| Metric | Observed value |
|---|---:|
| Total DNS packets | 2 |
| Queries | 1 |
| Responses | 1 |
| A records counted | 17 |
| NS records counted | 6 |
| No-error responses | 2 DNS-message record views / 100% shown |
| Request-response pairs | 1 |
| Average request-response time | 6.95 ms |
| Retransmissions | 0 |
| Unsolicited responses | 0 |

The record-type counters include records present across the decoded DNS messages (including additional and authority sections), while the response packet itself reports 15 answer RRs, 6 authority RRs, and 2 additional RRs.

![DNS statistics](Screenshots/DNS_Screenshots/03_dns_statistics.png)

## Endpoint and protocol profile

| Endpoint | Packets | Total bytes | Sent | Received |
|---|---:|---:|---:|---:|
| `192.168.50.50` | 2 | 615 B | 1 packet / 75 B | 1 packet / 540 B |
| `192.168.0.1` | 2 | 615 B | 1 packet / 540 B | 1 packet / 75 B |

At the protocol level, the two packets total 615 bytes: Ethernet contributes 28 B, IPv4 40 B, UDP 16 B, and DNS 531 B (**86.3%** of bytes).

![DNS IPv4 endpoints](Screenshots/DNS_Screenshots/02_ipv4_endpoints_dns.png)

![DNS protocol hierarchy](Screenshots/DNS_Screenshots/01_protocol_hierarchy_dns.png)

## Important observation: non-standard UDP port

The observed traffic uses UDP port **65333** on the resolver side, not the conventional DNS port 53:

```text
Query:    192.168.50.50:65282 → 192.168.0.1:65333
Response: 192.168.0.1:65333   → 192.168.50.50:65282
```

Wireshark decodes both packets as DNS in this capture. A non-standard port is not inherently suspicious; lab configuration, port forwarding, or a Wireshark decode-as rule can account for it. In a production baseline, document the expected resolver port and investigate unexpected deviations in context.

## Baseline conclusion

This sample represents a successful, low-latency DNS exchange:

- One query and one matching response use the same transaction ID (`0x002b`).
- The queried hostname is `us.pool.ntp.org`, type A, class IN.
- The resolver returns `RCODE 0` / **No error**.
- The recorded request-response time is **6.946 ms** with no retransmissions shown.
- The payload is response-heavy (540 B response versus 75 B query), which aligns with multiple answer, authority, and additional records.

For future comparisons, investigate absent responses, transaction-ID mismatches, elevated latency, changed response codes, unusual record counts, or resolver-port changes that are not part of the expected environment.
