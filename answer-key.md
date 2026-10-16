# Answer key — ground truth for the KYC demo corpus

Not part of the 20-document ingest set. This is the design reference so we
can verify Document Intelligence's extraction against what we actually wrote,
and so we know what the Cypher/GDS moments should find. Keep this out of the
`documents/` folder.

## Clusters

**Cluster A — the shell ring** (all in Wilmington, DE, all registered agent
Sterling Corporate Services LLC, all at the same suite, all formed within an
8-month window in 2019-2020):
- Meridian Holdings LLC
- Northgate Ventures LLC
- Cobalt Trading Co
- Anchor Point Partners LLC (the loan applicant)
- Ashford Capital Partners LLC
- Vantage Point Estates LLC

**Cluster B — legitimate operating companies** (real, distinct operating
addresses; share registered agent Pacific Registered Agents Inc, which is
incidental/benign — a deliberate contrast to Cluster A's agent+address
combination):
- Foothill Manufacturing Inc
- Cascade Robotics Inc
- Bluewater Logistics Inc (wholly-owned subsidiary of Foothill)
- Timberline Foods Inc (fully standalone — no ties to anyone else, a control/noise node)

**Cluster C — clean, unrelated pair** (registered agent Keystone Registered
Agents Inc; no ties to A or B):
- Redwood Analytics LLC
- Summit Creative Group LLC

## Ownership graph (`OWNS`)

- Dorian Voss --60%--> Meridian Holdings LLC
- Dorian Voss --100%--> Vantage Point Estates LLC
- Cobalt Trading Co --30%--> Meridian Holdings LLC
- Meridian Holdings LLC --40%--> Northgate Ventures LLC
- Priya Nandakumar --15%--> Northgate Ventures LLC
- Northgate Ventures LLC --35%--> Cobalt Trading Co
- Cobalt Trading Co --51%--> Anchor Point Partners LLC
- Vantage Point Estates LLC --20%--> Anchor Point Partners LLC
- Cobalt Trading Co --45%--> Ashford Capital Partners LLC
- Meridian Holdings LLC --25%--> Ashford Capital Partners LLC
- Elena Marsh --70%--> Foothill Manufacturing Inc
- Elena Marsh --55%--> Cascade Robotics Inc
- Foothill Manufacturing Inc --100%--> Bluewater Logistics Inc
- Marcus Webb --100%--> Timberline Foods Inc
- Ines Alvarez --80%--> Redwood Analytics LLC
- Ines Alvarez --65%--> Summit Creative Group LLC

**The cycle:** Meridian Holdings LLC → Northgate Ventures LLC → Cobalt
Trading Co → Meridian Holdings LLC (40% / 35% / 30%).

**The UBO story (Anchor Point Partners LLC, the loan applicant):** Dorian
Voss never appears in Anchor Point's own KYC packet, but controls it via two
independent paths:
1. Dorian Voss → Meridian Holdings LLC (60%) → Northgate Ventures LLC (40%) → Cobalt Trading Co (35%) → Anchor Point Partners LLC (51%) — 4 hops, through the cycle
2. Dorian Voss → Vantage Point Estates LLC (100%) → Anchor Point Partners LLC (20%) — 2 hops

**The bridge:** Priya Nandakumar is a director of Northgate Ventures LLC
(Cluster A, plus a 15% owner) and *also* a director of Cascade Robotics Inc
(Cluster B). That single `OFFICER_OF` edge is the only connection between
Cluster A and Cluster B — the betweenness centrality target.

## Officers / directors (`OFFICER_OF`)

- Dorian Voss — Manager, Meridian Holdings LLC; Manager, Vantage Point Estates LLC
- Priya Nandakumar — Director, Northgate Ventures LLC; Director, Cascade Robotics Inc
- Radu Costin — CFO, Cobalt Trading Co (no ownership stake — headcount/realism only)
- Elena Marsh — CEO, Foothill Manufacturing Inc; Chairperson, Cascade Robotics Inc
- Marcus Webb — President, Timberline Foods Inc
- Ines Alvarez — Managing Member, Redwood Analytics LLC; Managing Member, Summit Creative Group LLC

## Loan applications (`REQUESTS`, `GUARANTEES`)

1. Anchor Point Partners LLC — guarantors: Dorian Voss (personal), Cobalt Trading Co (corporate)
2. Northgate Ventures LLC — guarantor: Meridian Holdings LLC (corporate) — reinforces the cycle in a financial context
3. Foothill Manufacturing Inc — guarantor: Elena Marsh (personal) — the "boring, normal" file for contrast
4. Cascade Robotics Inc — guarantors: Elena Marsh (personal), Priya Nandakumar (personal) — puts the bridge person in a financial document too

## Screening / adverse media (`FLAGGED_BY`, `MENTIONED_IN`)

1. Dorian Voss — PEP hit (former senior economic advisor to a foreign Ministry of Finance, 2014-2018)
2. Meridian Holdings LLC — sanctions watchlist "possible match" (name similarity only), analyst notes the shared-suite pattern with other flagged shells
3. Media: regional business-journal piece on the Wilmington suite as a registration hub, names Sterling Corporate Services LLC, Meridian Holdings LLC, Cobalt Trading Co
4. Media: older piece naming Dorian Voss in a foreign asset-disclosure review (no charges, closed)

## What each demo beat should surface

- **UBO traversal** → both paths from Dorian Voss to Anchor Point Partners LLC
- **Cycle detection** → Meridian → Northgate → Cobalt → Meridian
- **Louvain** → Cluster A (6 companies) as one tight community, distinct from Cluster B and Cluster C; Timberline Foods Inc likely falls out as its own singleton
- **Betweenness centrality** → Priya Nandakumar as the highest-betweenness node (sole bridge between A and B)
- **PageRank** → Dorian Voss and/or Cobalt Trading Co ranking highest (multiple inbound high-weight ownership paths)
