# network_map

Turn an nmap XML scan into a single static HTML page with an interactive network graph. No server needed: you open the file in a browser.

Status: planning. v1 renders a single scan snapshot. Work is tracked in [Issues](../../issues).

## Pipeline

```
nmap XML  ->  normalized hosts (Layer 1)  ->  derived graph (Layer 2)  ->  one self-contained HTML (Cytoscape.js inlined)
```

The renderer only reads Layer 2. Derivation (subnets, edges, roles) is kept out of the parser.

## Recommended scan

```sh
sudo nmap -sV -O --traceroute -oX scan.xml 192.168.1.0/24
```

Without `--traceroute` there are no hop edges, and every host hangs off the gateway in a star. `-sV -O` feeds the service and OS badges.

## Data model

```ts
// Layer 1: normalized facts from the XML
type Scan = { id: string; startedAt: string; args: string; nmapVersion: string };

type Host = {
  id: string;              // MAC when on-link, else IP
  ips: string[];
  mac?: string;
  vendor?: string;
  hostnames: string[];
  status: "up" | "down";
  os?: { name: string; family?: string; accuracy: number };
  ports: Port[];
  hops?: Hop[];            // only with --traceroute
  seenIn: string[];
};

type Port = {
  port: number; proto: "tcp" | "udp"; state: string;
  service?: string; product?: string; version?: string;
  cpe: string[]; scripts: { id: string; output: string }[];
};

type Hop = { ttl: number; ip: string; rtt?: number; host?: string };

// Layer 2: derived, consumed by the renderer
type Subnet = { cidr: string; gatewayId?: string; hostIds: string[] };
type Edge   = { from: string; to: string; kind: "hop" | "gateway" | "member"; rtt?: number };
type Node   = Host & { role: Role; phantom?: boolean };   // phantom = traceroute hop never scanned
type Role   = "router" | "server" | "workstation" | "printer" | "iot" | "unknown";

type Graph = { scans: Scan[]; subnets: Subnet[]; nodes: Node[]; edges: Edge[] };
```

## Stack

Bun + TypeScript, Cytoscape.js inlined into the output page.

## Scope

Only scan networks you own or have written permission to test.
