# Map Selection Strategy and lookup order for IP and Port Filtering

* **State**: Draft
* **Date**: 2025-12-15

## Context and Problem Statement

The XDP program requires to both look at ports and ip addresses when it comes to allow or deny a packet with static rules.
Problem statement is using wrong lookup order and/or wrong data structures may waste both RAM and CPU resources.


## Decision Drivers

* **CIDR Support**: The data structure must natively support Longest Prefix Matching to handle subnets instead of static ip addresses.
* **Performance**: Lookup operations must be deterministic and extremely fast (O(1) or close to it).
* **Memory Efficiency**: The solution should not waste RAM, but performance takes precedence where trade-offs exist.

## Considered Options
### For Lookup Order 
* **Option 1**: `Checking Port First`
* **Option 2**; `Checking Source IP First`
### For IP Filtering
* **Option 1**: `BPF_MAP_TYPE_HASH` 
* **Option 2**: `BPF_MAP_TYPE_LPM_TRIE` 
* **Option 3**: `BPF_MAP_TYPE_ARRAY` 

### For Port Filtering
* **Option 1**: `BPF_MAP_TYPE_HASH`
* **Option 2**: `BPF_MAP_TYPE_ARRAY` (Size 65536)

## Decision Outcome
1:**`Lookup Order` Port first.**
2:**`BPF_MAP_TYPE_LPM_TRIE` for IP Addresses.**
3:**`BPF_MAP_TYPE_ARRAY` (Max Entries: 65536) for Ports.**

### Justification

**For IPs:**
LPM is selected because it's already a native solution for IP addresses when it comes to data structures.If an example is wanted to be given,LPM will stop using resources after 16 bits it's a /16 subnet network.
Also,It'll be a lot easier to add rules both at configuration and control space.
**For Ports:**
Since Ports already have a fixed key space and there is not many ports to use practically,Hash Map is not needed to prevent overhead.Array is selected for accessing a rule with a single CPU instruction.
**For Lookup Order:**
Looking port is selected because it's o(1) and LPM is o(32) or 0(128),and for dropping port scan packets as soon as possible.
### Positive Impacts

*   **Subnet Support**: We can now allow entire networks with single rules.
*   **Maximized Performance**: Port lookups are now the fastest possible operation in computer science.
*   **Simplicity**: The logic for checking a port becomes a simple array index read.
*   **Fail Fast**: Lookup order logic is preventing calculating 0(32) or 0(128) if the port is not allowed,ends up dropping with 0(1).