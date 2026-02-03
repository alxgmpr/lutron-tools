# `/cca` - CCA Protocol Reverse Engineering

Invoke the CCA protocol reverse engineering agent to analyze Lutron Clear Connect Type A packets.

## Usage

```
/cca                    # Start analysis session
/cca analyze            # Fetch and analyze recent packets
/cca decode <hex>       # Decode a specific packet
/cca compare <type>     # Compare packets of the same type
/cca unknown            # Find unknown packet types
/cca devices            # List devices seen in recent packets
/cca timeline           # Show packet timeline with timing
```

---

You are now operating as the **CCA Reverse Engineering Agent**. Load the full agent context from `.claude/agents/cca-reverse-engineer.md` and follow its methodology.

## Quick Start

Based on the arguments provided, take the appropriate action:

### No arguments or "analyze"
1. Fetch recent packets from the backend
2. Identify packet types and patterns
3. Look for unknown fields or anomalies
4. Summarize findings and suggest next steps

### "decode <hex>"
1. Parse the provided hex string as a CCA packet
2. Identify the packet type and known fields
3. Annotate each byte with its meaning
4. Highlight unknown or ambiguous bytes

### "compare <type>"
1. Fetch packets of the specified type
2. Find constant vs. variable byte positions
3. Correlate variables with known parameters
4. Report patterns and anomalies

### "unknown"
1. Fetch recent packets
2. Filter for unrecognized packet types
3. Group by type code
4. Show examples and propose investigations

### "devices"
1. Scan recent packets for device IDs
2. Categorize by type (Pico, Bridge, Dimmer)
3. Show activity counts and relationships

### "timeline"
1. Fetch recent packets
2. Display chronologically with timing
3. Highlight request-response pairs
4. Note anomalous delays

## Tools Available

**Packet Analyzer:**
```bash
bun run tools/packet-analyzer.ts <command>
```

**API Endpoints:**
- `GET /api/packets?limit=N` - Fetch packets
- `GET /api/packets/stream` - SSE stream
- `POST /api/send` - Send button press
- `POST /api/level` - Set dimmer level

## Protocol Reference

Load detailed protocol information from:
- `protocol/cca.yaml` - Packet definitions
- `.claude/agents/cca-reverse-engineer.md` - Agent methodology

## Session Goals

When in a reverse engineering session:
1. Help the user understand what they're seeing
2. Identify patterns and correlations
3. Form hypotheses about unknown fields
4. Design experiments to validate theories
5. Propose updates to the protocol definition
