# CCNA 2.0 — Instructor Professional Development (IPD)

Hands-on [Cisco Modeling Labs (CML)](https://developer.cisco.com/modeling-labs/) topologies
and instructor documentation for the **Implementing and Administering Cisco Solutions
(200-301 CCNA) v2.0** exam blueprint, organized by exam domain.

Each domain folder is a self-contained teaching lab: an importable CML topology, per-device
startup configurations, and instructor-facing walkthroughs with validated `show`-command
output and "forcing functions" for live demonstration.

## Domains

| # | Domain | Weight | Status |
|:-:|--------|:------:|--------|
| 1 | [Network Infrastructure and Connectivity](domain-1/) | 25% | 🚧 Placeholder |
| 2 | [Switching and Network Access](domain-2/) | 25% | 🚧 Placeholder |
| 3 | [IP Routing](domain-3/) | 20% | ✅ Available |
| 4 | [Network Services and Security](domain-4/) | 20% | 🚧 Placeholder |
| 5 | [AI, and Network Operations and Management](domain-5/) | 10% | 🚧 Placeholder |

> Domain titles and weights follow the official *Implementing and Administering Cisco
> Solutions (200-301 CCNA) v2.0* exam topics.

## What's in each domain folder

```
domain-N/
├── README.md                     # domain overview, exam-topic mapping, diagram, addressing
├── <topology>.yaml               # importable CML lab (nodes, links, embedded configs)
├── config/                       # per-device startup configurations
└── docs/                         # instructor walkthroughs, one per exam sub-topic
```

## Getting started

1. Pick a domain folder above and open its `README.md`.
2. Import the domain's `.yaml` into CML (**Import → from file**), then start the lab —
   or build it by hand from the files in `config/`.
3. Work through the walkthroughs in that domain's `docs/` directory.

## Status

Domain 3.0 (IP Routing) is complete and validated on CML. The remaining domains are
placeholders and will be added over time.

---

*CCNA and Cisco Modeling Labs are trademarks of Cisco Systems, Inc.*
