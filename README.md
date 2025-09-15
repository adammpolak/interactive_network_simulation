Interactive Network Simulation – Architecture and Authoring Guide

Overview
This project simulates the journey of packets across hosts, NICs, media converters and fiber/copper media. It visualizes OSI layers, interfaces, physical media, logs, animations, and step-by-step highlights.

Core Concepts
- Scenarios: High-level stories (Direct 1Gbps Copper, Direct 100Mbps Copper, Fiber 1Gbps SX). Each scenario is a list of scenes, and each scene is a list of steps.
- Steps: Each step renders a log item (title + subSteps) and executes a flow (send path, optional media hop, optional receive path) that highlights layers and interfaces.
- Layers and Interfaces: Each host has layers (app, ip, mac, phy) and interfaces (if_app_ip, if_ip_mac, if_mac_phy). Fiber adds MC↔SFP and optical interfaces.
- Media: copper (twisted pair) and fiber. Animations render voltage pulses (copper) or optical pulses (future).

Declarative Flow DSL
We express layer traversals using a small flow engine backed by a topology graph.

Topology graph
- Nodes: 'A.ip', 'A.if_ip_mac', 'A.mac', 'A.if_mac_phy', 'A.phy', 'medium', 'B.phy', 'B.if_mac_phy', 'B.mac', 'B.if_ip_mac', 'B.ip'.
- Edges: Valid adjacencies between nodes (e.g., 'A.ip' ↔ 'A.if_ip_mac', 'A.phy' ↔ 'medium', 'medium' ↔ 'B.phy').

Flow contract (pseudo-type)
flow({
  send:    { start: 'A.ip', end: 'A.phy' },
  medium:  'copper' | 'fiber' | undefined,
  receive: { start: 'B.phy', end: 'B.ip' } | undefined,
  sendDetail: { [nodeId]: string },   // interface detail text per node
  recvDetail: { [nodeId]: string },
  stepDelay: number
})

Engine behavior
1) Derives a path with BFS from start → end using the topology graph. If no path, throws (prevents skipped layers).
2) Highlights each node in order with step delay; applies detail text to interfaces.
3) If medium specified, runs appropriate media animation.
4) If receive specified, derives a second path and repeats for the other host.

Why this prevents mistakes
- Continuity: Paths must exist in the graph (no “teleporting” from IP to PHY). If a node is missing or ordering is wrong, the step fails fast.
- Completeness: Interface nodes in the path can (optionally) require detail text. Missing detail becomes immediately obvious.
- Single source of truth: Logs, highlights, and animations are all driven by the flow. We don’t hand-code sequences per step anymore.

Authoring Steps
Each step includes:
- Log: { title, subSteps[] }
- Flow: flow({ send, medium, receive, sendDetail, recvDetail, stepDelay })

Example (ARP broadcast)
log: "ARP Request: Who has 192.168.1.11?"
flow({
  send: { start: 'A.ip', end: 'A.phy' },
  medium: 'copper',
  receive: { start: 'B.phy', end: 'B.ip' },
  sendDetail: { 'A.if_ip_mac': 'TX descriptor', 'A.if_mac_phy': 'GMII → PHY' },
  recvDetail: { 'B.if_ip_mac': 'DMA RX + IRQ' },
  stepDelay: 180
});

Validating Scenarios
- Changes to the topology immediately impact all flows; invalid flows throw errors early.
- To add new hardware paths (e.g., MC↔SFP, SFP breakout pins), extend the topology with new nodes and edges; existing steps continue working while new steps can traverse the expanded graph.

Adding / Updating Scenarios
1) Add/modify a scenario’s scenes/steps in index.html under scenarioLibrary population.
2) For each new step:
   - Write log text (title + subSteps).
   - Define a flow with clear start/end and optional receive; supply interface detail text where appropriate.
   - If the path crosses media, specify medium.
3) Verify the step visually and ensure the log and highlights match.

Fiber / Media Converter Notes
- Fiber scenario includes additional interfaces (MC↔SFP MSA and optical TX/RX). Extend the topology with nodes like 'A.if_copper_mc', 'A.if_mc_sfp', 'if_sfp_fiber_A', etc., and add edges to connect host PHY to MC copper, MC to SFP, SFP to fiber.
- Once nodes are in the topology, flows will validate and highlight these interfaces in order.

Play / Step / Back
- Play loops through steps, invoking logs and flows in order.
- Next runs a single step.
- Prev rewinds by soft-resetting and replaying all prior steps, rebuilding the log and highlights strictly from history, ensuring consistency.

Best Practices
- Always express send and receive paths as flows with start/end nodes; do not manually toggle highlights in arbitrary order.
- Provide brief detail text on interface nodes (if_ip_mac, if_mac_phy, MC↔SFP) to maintain educational clarity.
- Keep log substeps aligned with flow hops; avoid ambiguous phrasing like “it is sent” without indicating by whom and along which interfaces.
- When adding new hardware segments, first update the topology nodes/edges, then update steps to traverse those nodes.

Roadmap
- Extend topology for full fiber path (MC copper PHY ↔ MC logic ↔ MC↔SFP MSA ↔ Optical TX/RX ↔ Fiber).
- Add scenario for SFP breakout board with I2C pins (SDA/SCL), MOD_DEF*, TX_DISABLE, TX_FAULT, RX_LOS; model power-up and RX_LOS behavior via topology.

Where to Look in Code
- Flow engine and topology: inline in index.html (search for "FLOW DSL").
- Scenario steps: scenarioLibrary population block in index.html.
- UI / DOM bindings and helpers: index.html script top.


