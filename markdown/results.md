# avro2neo: Loading avro-format transaction trees into neo4j and hence into other tools

## Introduction

ESI data is processed by `CorMel` into *transaction trees* with the following
structure:

* CorMel defines 5 "segments" in all: A, E, H, T and U, which store data
pertaining to transaction trees.
* The common features (across all entities of the transaction tree) are stored
in a `U` segment for each transaction tree, which acts as the root of the tree.
* The `U` node is connected to a single `T` node, which is the transaction that
spawns the other transactions in the tree.
* Each transaction `T` node can have 0 or more child transactions (`T` nodes).
* The leaf nodes can be `T` nodes, or alternatively hop (`H`) nodes, `A` or `E`
nodes.
* The `A` and `E` nodes often have blank properties and/or default data, and
so are loaded into neo4j but are not used for further analysis. They appear to
be descriptive and are shared between transaction trees.

Each CorMel segment is associated with a standard list of fields, hence it is
equivalent to a table in a relational database, or a node in a property graph.

Although transaction trees have a natural mapping into graph data structures,
this aspect of their nature has not been explored in Amadeus. The research
hypothesis is that transaction trees can be classified into two categories,
depending on whether they are associated with *successful* or *unsuccessful*
outcomes.

Note that *success* and *failure* have specific meanings in this context. For
example, a customer could decide to abandon a search, or even a booking, before
completing all steps. Because the customer made the decision, Amadeus
infrastructure was not at fault, and the incomplete transaction tree does not
represent a problem for the Amadeus devops team to fix[^1]. Another scenario
would be that a network link failure occurred, in which case the service
response never arrived, so the customer had no choice and was unable to proceed
with the transaction. As seen by the customer, the system became unresponsive,
or even provided an error message of the form "The system is currently
unavailable.  Please try again later." Neither of these represents a
*successful* outcome. Even worse, such problems are likely to recur, and to
affect other customers. Hence they need to be addressed as soon as possible.

[^1]: Perhaps the Customer Experience team might be interested in analysing
transactions abandoned by customers, as there might be ways of improving the
customer experience with the intention of increasing the completion rate of
transactions. However, such offline analysis is outside the scope of the
present study.
 
The research hypothesis is that there are *structural* and/or *property*
differences between successful and unsuccessful transaction trees. Furthermore,
it might even be possible to suggest some queries that could be used to derive
the root cause of *correlated* problems.

One possible structural difference might include highly unbalanced trees,
reflecting the fact that, if a particular edge in the tree failed, the
redundancy built into the Amadeus services infrastructure is such that other
parts of the transaction might succeed (and generate lots of activity in the
logs, hence segments in the CorMel transaction tree) although the overall
transaction might fail.

Property differences could be subtle and might be more useful for root cause
determination. For example, the transaction might invoke a service that was
recently updated, or be directed to an endpoint that is struggling to meet
demand. Such property data is stored in the segment records. (Graph) database
queries might help to find common property settings across problematic
transaction trees.

## The processing pipeline

More information can be found in the related implementation documentation.

A summary of the processing pipeline is shown in Figure 1 below. 

![Figure 1: Overview of the processing pipeline.](../graphics/pipeline-CROPPED.png){height=9cm}

The starting point is the `CorMel` system itself, from which an Avro-format
extract is taken on a timed basis and saved in a file, with one file per
extract run. The given example file contained 24499 transaction trees. A (Java)
program was written to upload this data to a `Neo4j` database, where it is
represented as a graph, with the `U`, `T`, `H`, `A` and `E` segments from
CorMel being stored as Neo4j nodes, linked together with Neo4j edges that
represent the *flow* of the transaction trees.

Samples of such data can be exported in Graphml format from Neo4j and stored in
(data-oriented, not graphics-oriented) files that are optimised for the `Gephi`
and `yEd` applications, where they can be visualised and analysed.

Within each of these applications, the nodes and edges are
assigned (`x`,`y`) coordinates according to the layout algorithms.  

It is easier to interpret transaction trees by highlighting them in the context of other
transaction trees, so a python program was written to generate modify the edge colour
so that highlighted transaction trees have edges coloured black and other trees have edges
that are light grey in colour.

It is also possible to run standard graph algorithms on the data in Neo4j, or alternatively
in `Gephi` or `yEd`. The results of such analysis are described below.

## Preliminary analysis results

It should be noted that the preliminary analysis results are based on a
sample data of 20 transaction trees, ignoring the `A` and `E` segment nodes.

The first interesting feature is that the number of components is 8 rather than
20 (the number of (logical) transaction trees), because of the presence of shared
nodes (and even edges).

If we compare highlighted tree #1 using yEd's `circular` and `treeBalloon` layouts,
we see that certain features are relatively stable between the two representations.

![Figure 2: Highlighted tree #1, for 20 tree sample, yEd's "circular" layout, annotated to show features in coloured boxes.](yed/annotated/sample20yed_circular_hl01.png){height=9cm}

The box on the right, with a pale green background, shows a relatively simple
arrangement of isolated trees, differing in size from 3 nodes (`U`, `T` and
`H`) to a much larger tree with many nodes. The box with the pale blue
background includes most nodes in transaction tree 1, except for a branch that
leads to a shared node. The box with the yellow background includes most nodes
of two trees, except for their own version of the branch that leads to the same
node that is shared with transaction tree 1. The box with the pale orange
background includes both simple and shared edges associated with a particular
node. There is a lot of complexity here, which is worthy of further study.

![Figure 3: Highlighted tree #1, for 20 tree sample, yEd's "treeBalloon" layout, annotated to show features in coloured boxes.](yed/annotated/sample20yed_treeBalloon_hl01.png){height=9cm}

Figure 2 and Figure 3 are very similar, with slightly different ways of showing
the more complex overlapping transaction trees.

Indeed, yEd is also able to display certain analytical properties of graphs.
For this study, the following graphs were considered interesting. Firstly, the
number of edges connected to each node (its *degree*) is a measure of how many
service endpoints are tasked with work from that node. If any of these service
endpoints fails to provide a response, the transaction tree could block at that
node. See Figure 4 below.

![Figure 4: 20 tree sample, yEd's "treeBalloon" layout, showing the (out)degree for each node.](yed/sample20yed_treeBalloon_numberConnectedEdges.png){height=9cm}

The node's degree provides a "local" measure of the work passing through that
node.  However, the node's position in the tree also affects the flow of data
and control. The *paths* of these flows should also be considered. One such
measure is the node's *betweenness centrality*, which is represented in Figure
5.

![Figure 5: 20 tree sample, yEd's "treeBalloon" layout, showing the betweenness centrality for each node.](yed/sample20yed_treeBalloon_nodeBetweennessCentrality.png){height=9cm}

The ranking of nodes (according to the computed metric) changes between Figures
4 and 5, reflecting the differing weighting of local and global information
relating to the transaction tree(s) containing that node.

By contrast with yEd, Gephi provides *reports* on various conceptually-linked
graph metrics in the form of HTML pages referencing PNG plots of those metrics.

Figure 6 below indicates that there are 3 components with very few nodes, but
with other transaction trees having significantly more nodes (~300 nodes in one
case).  More analysis would be needed to determine whether the distribution of
component size might be a good predictor of whether a transaction tree has
succeeded or failed.

![Figure 6: 20 tree sample: the component size distribution for the 8 weakly connected components.](gephi/connectedComponentsReport/cc-size-distribution.png){height=9cm}

The graph *diameter* was found to be 9, with an average path length just
exceeding 3.  Therefore, a `U->T->T->H` path is average path through the
transaction trees in the sample. Perhaps the distribution of path lengths might
be a good predictor, but this is not computed by Gephi (although it could be
computed, with a little effort, in Neo4j).

Gephi provides plots of *betweenness centrality*, *closeness centrality* and
*eccentricity*, see Figures 7, 8 and 9 below.

![Figure 7: 20 tree sample: the Betweenness Centrality distribution.](gephi/graphDistanceReport/Betweenness Centrality Distribution.png){height=9cm}

![Figure 8: 20 tree sample: the Closeness Centrality distribution.](gephi/graphDistanceReport/Closeness Centrality Distribution.png){height=9cm}

![Figure 9: 20 tree sample: the Eccentricity distribution.](gephi/graphDistanceReport/Eccentricity Distribution.png){height=9cm}

In Figure 7, most of the mass of the Betweenness Centrality distribution can be
found near 0, but there is also a relatively long tail. In Figure 8, the
Closeness Centrality has two relatively common values (0 and 1) and the
remainder lie in between. According to Figure 9, Eccentricity takes values
between 0 and 9, with a preference for lower values.  It has arguably the
simplest of the three distributions and so might be easiest to compare between
successful and failed transaction trees.
 
## Next Steps

