# Finding the Ultimate Beneficial Owner in a Pile of KYC Documents

With Document Intelligence, you can build a knowledge graph from existing documents, including plain text files, PDFs, and Microsoft Word files, without writing a single line of code. Documents that previously sat unused and unqueried can now be searched and acted upon.

Once your documents are in a knowledge graph, you can programmatically uncover connections hidden across pages of text. Take uncovering the ultimate beneficial owner from know-your-customer compliance documents, for example. Previously, each document would need to be carefully pored over, and assuming a single reader even looked at the right documents, connections would have to be discovered on the fly. That's how bad actors conceal fraud or illicit payments.

## What we're looking at today

A bank's compliance team receives a stack of onboarding paperwork for a new corporate client, **Anchor Point Partners**, applying for a loan: registration filings, ownership disclosures, board minutes, a loan application, a sanctions screening note, a couple of news clippings. 

The compliance team pulled twenty documents to screen Anchor Point Partners, including some related companies' previous loan applications. 

Read individually, every document looks routine. However, two claims seemed a bit suspicious to the compliance team:

From the loan application for Anchor Point Partners LLC:

> Guarantor Dorian Voss is disclosed as an individual guarantor
> providing personal support for the facility; his relationship to the
> applicant entity is not further specified on this application. Applicant's
> own cap table does not list Mr. Voss as a direct owner.

And from the loan application for Cascade Robotics Inc:

> Guarantor Priya Nandakumar is a director of the applicant with no equity
> interest; her limited guarantee reflects her board role rather than an
> ownership stake.

Read at face value, everything seems to be in order, but overlapping ownership interests can make knowing who your true customer is a difficult and confusing mess.

Regulators call this the ultimate beneficial owner (UBO): the real person at the top of an ownership chain, however many shell companies and holding entities sit between them and the entity that showed up asking for a loan. Answering it by hand means manually tracing ownership percentages across a dozen documents and hoping you don't miss a layer.

## Document Intelligence makes building a knowledge graph easy

You can create a knowledge graph from your documents easily with Document Intelligence. Start by staging your documents into Document Intelligence. Then click "Generate Model" and put in a prompt to guide the LLM:

![11_generate_model_final](/Users/corydonbaylor/Documents/blogs/kyc_doc_intel/11_generate_model_final.gif)

Using the conversational agent, you can simply say what changes you would like in the graph model:

![12_rename](/Users/corydonbaylor/Documents/blogs/kyc_doc_intel/12_rename.gif)

Document Intelligence makes it easy to understand your new knowledge graph. You can view both the graph model and a preview of the graph once its populated:

![13_visualize](/Users/corydonbaylor/Documents/blogs/kyc_doc_intel/13_visualize.gif)

Once you are happy with your graph model, simply import it into an empty AuraDB.

## The documents, the backup, and the notebooks

First, some housekeeping before you can start analyzing your new knowledge graph. 

Everything is on GitHub. That includes the 20 source documents themselves, an `answer-key.md` with the ground truth we built the scenario around, a database backup you can restore into your own Aura instance to reproduce this from scratch, and two notebooks.

We used `demo.ipynb` to do some entity resolution on the Document Intelligence output, merging duplicate mentions into one canonical node per entity. That took the graph from 51 Organization nodes down to the real 14.

This post walks through every code cell in the second notebook, [`demo_part2.ipynb`](demo_part2.ipynb), where we took that cleaned-up graph and went looking for the UBO. We use plain Cypher where a pattern match is all you need, and Aura Graph Analytics (Neo4j's on-demand Graph Data Science product) where the question requires a real algorithm.

## Setting up: two connections, not one

The notebook opens two separate connections, because this Aura instance runs Graph Data Science as an on-demand session rather than as procedures built into the database:

```python
driver = GraphDatabase.driver(URI, auth=(USERNAME, PASSWORD), notifications_min_severity="OFF")
driver.verify_connectivity()

api_credentials = AuraAPICredentials(
    client_id=os.environ["CLIENT_ID"],
    client_secret=os.environ["CLIENT_SECRET"],
    project_id=os.environ.get("TENANT_ID") or None,
)
sessions = GdsSessions(api_credentials=api_credentials)
db_connection = DbmsConnectionInfo(uri=URI, username=USERNAME, password=PASSWORD)

gds = sessions.get_or_create(
    session_name=SESSION_NAME,
    memory=SessionMemory.m_2GB,
    db_connection=db_connection,
    ttl=timedelta(minutes=30),
    neo4j_driver_config={"notifications_min_severity": "OFF"},
)
gds.verify_connectivity()
```

driver is a standard Neo4j Python driver we use for all plain Cypher queries, and gds is a separate Graph Data Science session we use to run graph analytics against the same AuraDB instance.

## Tracing the real owner

The loan applicant in our document set is **Anchor Point Partners LLC**. Its own KYC packet lists exactly two owners, both corporate entities: Cobalt Trading Co and Vantage Point Estates LLC. No individual person is named anywhere on its own paperwork.

A variable-length Cypher traversal on the `OWNS` relationship walks past those immediate corporate owners to find who actually controls the company, however many layers deep that person sits:

```cypher
MATCH p = (person:Person)-[:OWNS*1..6]->(applicant:Organization {legalName: "Anchor Point Partners LLC"})
RETURN person.name AS beneficialOwner,
       [n IN nodes(p) | coalesce(n.legalName, n.name)] AS path,
       length(p) AS hops
ORDER BY hops
```

The result: **Dorian Voss** reaches the loan applicant through two independent paths.

- A short one, two hops: Dorian Voss → Vantage Point Estates LLC (100% owned) → Anchor Point Partners LLC (20%)
- A longer one, four hops: Dorian Voss → Meridian Holdings LLC (60% owned) → Northgate Ventures LLC (40%) → Cobalt Trading Co (35%) → Anchor Point Partners LLC (51%)

Anchor Point had claimed that he was just a guaranteer, but he's the actual controlling owner, reachable only by walking the graph.

To see it rather than just read it, the next cell reruns the same shape of query, but returns the path itself instead of extracted fields, and hands that straight to `neo4j_viz`:

```python
from neo4j_viz.neo4j import from_neo4j

ubo_graph_result = driver.execute_query(
    """
    MATCH p = (person:Person)-[:OWNS*1..6]->(applicant:Organization {legalName: "Anchor Point Partners LLC"})
    RETURN p
    """,
    database_=DATABASE,
)

VG_ubo = from_neo4j(ubo_graph_result)
for node in VG_ubo.nodes:
    node.caption = node.properties.get("legalName") or node.properties.get("name")
    node.color = "#4C72B0" if "Person" in node.properties.get("labels", []) else "#DD8452"
for rel in VG_ubo.relationships:
    rel.caption = f"OWNS {rel.properties.get('pct', '')}"

VG_ubo.render_widget(initial_zoom=1.0)
```

![1_real_owner](/Users/corydonbaylor/Documents/blogs/kyc_doc_intel/1_real_owner.png)

## Finding the loop

While we're in traversal territory, one more pattern match: does any company's ownership chain loop back to itself? Worth checking here specifically, since Meridian Holdings LLC, the company Dorian Voss directly owns 60% of and the first link in his longer path to Anchor Point Partners LLC, is also one of three companies that show up owning each other in a circle.

```cypher
MATCH p = (o:Organization)-[:OWNS*2..6]->(o)
RETURN DISTINCT [n IN nodes(p) | n.legalName] AS cycle, length(p) AS length
```

It does: Dorian Voss's own holding company, Meridian Holdings LLC, owns Northgate Ventures LLC, which owns Cobalt Trading Co, which owns Meridian Holdings LLC back again, a closed three-company loop (40%, 35%, and 30% stakes respectively). Circular ownership is a textbook obfuscation red flag in AML/KYC work, again found with a pattern match, not an algorithm.

Same visualization approach as before, this time capped with `LIMIT 1` since a cyclic path can be returned starting from any of its three nodes and we only need to see it once:

```python
cycle_graph_result = driver.execute_query(
    """
    MATCH p = (o:Organization)-[:OWNS*2..6]->(o)
    RETURN p
    LIMIT 1
    """,
    database_=DATABASE,
)

VG_cycle = from_neo4j(cycle_graph_result)
for node in VG_cycle.nodes:
    node.caption = node.properties.get("legalName") or node.properties.get("name")
for rel in VG_cycle.relationships:
    rel.caption = f"OWNS {rel.properties.get('pct', '')}"

VG_cycle.render()
```

![2_loop](/Users/corydonbaylor/Documents/blogs/kyc_doc_intel/2_loop.png)

## Louvain: the ring isn't one blob, it's two

Some questions require more than just a query. Community detection asks: are there clusters of densely-interconnected entities in this graph that nobody explicitly described as a group? That's a global optimization problem (maximizing modularity across the whole graph), not a traversal. There's no pattern to match because nobody specified what the pattern should look like.

Running an algorithm on the `gds` session first requires projecting a graph into it. You can do this by writing a cypher query for just the subset of the graph that you want to examine:

```python
if gds.graph.exists("ownershipGraph"):
    gds.graph.drop("ownershipGraph")

G, project_result = gds.graph.project.cypher(
    "ownershipGraph",
    """
    CALL () {
        MATCH (a)-[:OWNS]->(b)
        RETURN a AS source, b AS target
        UNION
        MATCH (a)-[:OWNS]->(b)
        RETURN b AS source, a AS target
        UNION
        MATCH (a)-[:OFFICER_OF]->(b)
        RETURN a AS source, b AS target
        UNION
        MATCH (a)-[:OFFICER_OF]->(b)
        RETURN b AS source, a AS target
    }
    RETURN gds.graph.project.remote(source, target, {
      sourceNodeLabels: labels(source),
      targetNodeLabels: labels(target),
      relationshipType: 'CONNECTED'
    })
    """,
)
print(f"Projected {G.node_count()} nodes, {G.relationship_count()} edges (both directions)")
```

Two things worth calling out in that query. First, it includes both `OWNS` and `OFFICER_OF` relationships: for community detection, we want "who is structurally connected to whom," regardless of whether the connection is an ownership stake or a board seat. Second, each `MATCH` appears twice, once forward and once with `source`/`target` swapped. Ownership and officer relationships point in a direction in the data, but for finding clusters we want the algorithm to treat a connection as a connection regardless of direction, so we feed it both directions explicitly.

With the graph projected, running Louvain is one line. One thing to flag before you run it: Louvain finds communities across the whole projected graph, all 19 nodes, not just the six companies connected to Dorian Voss, so the raw count below will be higher than the "two" in this section's title. What we're actually watching for is which of those communities those six companies land in:

```python
louvain_result = gds.louvain.mutate(G, mutate_property="louvainCommunity")
print(f"Louvain found {louvain_result.community_count} communities "
      f"(modularity {louvain_result.modularity:.3f})")

louvain_stream = gds.graph.node_properties.stream(
    G, ["louvainCommunity"], db_node_properties=["legalName", "name"]
)
louvain_stream["entity"] = louvain_stream["legalName"].fillna(louvain_stream["name"])
louvain_stream = louvain_stream[["entity", "louvainCommunity"]].sort_values(
    ["louvainCommunity", "entity"]
)
louvain_stream
```

`mutate` runs the algorithm and stores the result as a property on the in-session graph, without writing anything back to AuraDB. `gds.graph.node_properties.stream` then pulls that property back out as a table, and `db_node_properties` lets us ask for the original `legalName`/`name` fields from the database in the same call, so the result is readable without a separate lookup. The rest is pandas: coalesce the two possible name fields into one `entity` column and sort for display.

A quick groupby turns that table into the clusters directly:

```python
for community_id, members in louvain_stream.groupby("louvainCommunity")["entity"]:
    print(f"Community {community_id}: {', '.join(members)}")
```

Going in, we expected the six companies connected to Dorian Voss, both his direct holdings and the ownership loop they sit inside, to fall out as one tidy "ring" community. Louvain found something more interesting: it split them into **two** tighter sub-communities instead of one.

- `{Meridian Holdings LLC, Northgate Ventures LLC, Cobalt Trading Co, Ashford Capital Partners LLC, Radu Costin}`: the reciprocal-ownership core (the loop, plus Cobalt's CFO)
- `{Anchor Point Partners LLC, Vantage Point Estates LLC, Dorian Voss, Ministry of Finance}`: Dorian Voss's personal holding chain

The algorithm found real structure: Voss's direct personal holdings are genuinely more tightly connected to *him* than to the Meridian/Northgate/Cobalt loop, even though a corporate ownership stake (Cobalt's 51% of Anchor Point) is what connects the two groups. Both sub-communities stay clearly separate from the legitimate operating companies and the unrelated clean pair in our document set.

Last step for this section, visualize the projected graph colored by the community property we just computed:

```python
from neo4j_viz.gds import from_gds

VG = from_gds(gds, G, node_properties=["louvainCommunity"], db_node_properties=["legalName", "name"])
for node in VG.nodes:
    node.caption = node.properties.get("legalName") or node.properties.get("name")
VG.color_nodes(property="louvainCommunity")
VG.render_widget(initial_zoom=0.8)
```

![3_louvain](/Users/corydonbaylor/Documents/blogs/kyc_doc_intel/3_louvain.png)

*[Screenshot / visualization: the graph colored by Louvain community]*

## Betweenness: the bridge

Betweenness centrality requires computing shortest paths between every pair of nodes in the graph and counting how often each node sits on one of them. The projected graph `G` is already sitting in the session, so there's no second projection step:

```python
betweenness_result = gds.betweenness.mutate(G, mutate_property="betweenness")
print(f"Betweenness computed for {betweenness_result.node_properties_written} nodes")

betweenness_stream = gds.graph.node_properties.stream(
    G, ["betweenness"], db_node_properties=["legalName", "name"]
)
betweenness_stream["entity"] = betweenness_stream["legalName"].fillna(betweenness_stream["name"])
betweenness_stream = betweenness_stream[["entity", "betweenness"]].sort_values(
    "betweenness", ascending=False
)
betweenness_stream.head(10)
```

Same `mutate` plus `stream` pattern as Louvain, just with a different algorithm and property name. The result, sorted and limited to the top 10:

| Entity | Betweenness |
|---|---|
| Northgate Ventures LLC | 80.0 |
| Priya Nandakumar | 72.0 |
| Cascade Robotics Inc | 60.0 |
| Cobalt Trading Co | 53.0 |
| Meridian Holdings LLC | 49.0 |

**Northgate Ventures LLC** scores highest. It sits in the middle of the ownership loop. The interesting result is who comes second: **Priya Nandakumar**, the one *person* near the top of an otherwise all-company ranking.

Priya Nandakumar is a director of Northgate Ventures LLC, part of the ring, and *also* a director of Cascade Robotics Inc, a legitimate operating company with no ownership ties to anyone in the ring. Louvain actually placed her fully inside the legitimate cluster, grouped with Cascade Robotics, Foothill Manufacturing, Bluewater Logistics, and Elena Marsh. Community detection alone would tell you she belongs with the clean companies. Betweenness centrality is what reveals she's also the single connection carrying the ring into a legitimate boardroom. Two algorithms, asked two different questions, and you need both answers to see the full picture.

The final visualization puts both results on one graph:

```python
VG_bw = from_gds(
    gds, G,
    node_properties=["louvainCommunity", "betweenness"],
    db_node_properties=["legalName", "name"],
)
for node in VG_bw.nodes:
    node.caption = node.properties.get("legalName") or node.properties.get("name")
VG_bw.color_nodes(property="louvainCommunity")
VG_bw.resize_nodes(property="betweenness")
VG_bw.render_widget(initial_zoom=0.8)
```

![visualisation(10)](/Users/corydonbaylor/Documents/blogs/kyc_doc_intel/visualisation(10).png)

Pulling both `louvainCommunity` and `betweenness` into the same `from_gds` call means one visualization can carry both results at once: `color_nodes` shows which community each entity landed in, and `resize_nodes` scales each node by its betweenness score. Priya Nandakumar should show up as a small node, since she isn't a hub by direct connection count, sized clearly larger than the rest of her community. That combination, small by degree and large by betweenness, is the visual signature of a bridge: unremarkable locally, structurally important globally.

![4_betweeness](/Users/corydonbaylor/Documents/blogs/kyc_doc_intel/4_betweeness.png)

## Cleaning up

The notebook's last cell closes out the session:

```python
G.drop()
sessions.delete(session_name=SESSION_NAME)
print("Session deleted:", SESSION_NAME)
```

An Aura Graph Analytics session is a billed resource for as long as it exists, so we drop the projected graph and delete the session explicitly rather than relying on the `ttl` timeout to clean it up for us.

## The takeaway

On paper, this file looked clean. Anchor Point Partners LLC's own cap table named no individual owner, and its loan application said Dorian Voss's relationship to the company was "not further specified." Priya Nandakumar's guarantee was filed as routine board-role disclosure, with her outside directorship at Northgate Ventures LLC noted as having "no bearing" on the loan she was guaranteeing.

The graph showed something else. Dorian Voss controls Anchor Point Partners LLC through two independent ownership paths, despite never appearing on its own paperwork. Those paths run through a closed ownership loop: Meridian Holdings LLC, Northgate Ventures LLC, and Cobalt Trading Co own each other in a circle. Louvain split that structure into two tightly connected sub-communities, both clearly separate from the legitimate companies in the document set. And betweenness centrality showed that Priya Nandakumar's "no bearing" board seat is in fact the single connection carrying that shell network into a legitimate company's boardroom.

None of this was stated anywhere in the source documents. Document Intelligence turned 20 files into a graph without us writing an extraction rule. Entity resolution with GDS turned fragmented mentions into one canonical graph. And from there, a mix of plain Cypher traversal and two GDS algorithms surfaced facts that only exist as a property of the graph as a whole, not of any single document in the pile.

Full writeup of the schema, the document corpus design, and the entity resolution approach: **[link to repo]**.
