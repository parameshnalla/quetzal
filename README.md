Quetzal (*Que*ry Tran*z*l*a*tion *L*ibraries)
=======

SPARQL to SQL translation engine for multiple backends, such as DB2, PostgreSQL and Apache Spark.   

# Philosophy
The goal of Quetzal is to provide researchers with a framework to experiment with various techniques to store and query graph data efficiently.  To this end, we provide 3 modular components that:
* Store data:  In the current implementation, data is stored in using a schema similar to the one described in [SIGMOD 2013 paper](http://dl.acm.org/citation.cfm?id=2463718).  The schema lays out all outgoing (or incoming) labeled edges of a given vertex *based on the analysis of data characteristics* to optimize storage for a given dataset.  The goal in the layout is to store the data for a given vertex on a single row in table to optimize for STAR queries which are very common in SPARQL.  
* Compile SPARQL to SQL:  In the current implementation, given a set of statistics about the dataset's characteristics, the compiler can compile SPARQL 1.1 queries into SQL.  The compiler will optimize the order in which it executes the SPARQL query based on statistics of the dataset.
* Support for SQL on multiple backends:  In the current implementation, we support DB2, PostgreSQL, and Apache Spark.  The first two are useful for workloads that require characteristics normally supported by relational backends (e.g., transactional support), the third targets analytic workloads that might mix graph analytic workloads with declarative query workloads. 

# Overview of Components
* Data Layout:  The current implementation uses a *row based layout of graph data*, such that each vertex's incoming edges or outgoing edges are laid out as much as possible on the same row.  For a detailed set of experiments that examine when this layout is advantageous, see [SIGMOD 2013 paper](http://dl.acm.org/citation.cfm?id=2463718).  Outgoing edges are stored in a table called DPH (direct primary hashtable), and incoming edges are stored in a table called RPH (reverse primary hashtable).  Because RDF can have many thousand properties, dedicating a column per property is not an option (in fact, some datasets will exhaust most database systems limits on the number of columns).  RDF data is sparse though, so each vertex tends to have a small subset of the total number of properties.  The current implementation performs an analysis of which properties co-occur with which others, and uses graph coloring to build a hash function that maps properties to columns.  Properties that co-occur together are typically not assigned to the same row.  If they do get assigned to the same row because a single vertex has several hundred edges to all sorts of properties, then collisions are possible and the schema records this fact, and the SQL is adjusted appropriately.  Note that for multi-valued properties, DPH and RPH record only the existence of the property for a given vertex, actual values require a join with a DS (direct secondary) and RS (reverse secondary) table, respectively.
* SPARQL-SQL compiler:  In the current implementation, this compilation job is done by a class called ``com.ibm.research.rdf.store.sparql11.planner.Planner``, in a method called ``public Plan plan(Query q, Store store, SPARQLOptimizerStatistics stats)``.  The goal of the planner is to compile the SPARQL query into SQL, *re-ordering* the query in order to start with the most selective triples (triples with the least cost), joining it with the second most selective triple based on what becomes available when one evaluates the first triple, and so on.  In doing so, the planner must respect the semantics of SPARQL (e.g., not join two variables that are named the same but are on two separate brances of a UNION).  The Planner employs a greedy algorithm to evaluate what available nodes exist for planning, and which one should be planned first.  AND nodes get collapsed into a single "region" of QueryTriple nodes because any triples within an AND node can be targeted first.  Each triple node within an AND can evaluate its cost based on what variables are available, and each node has a notion of what variables it can produce bindings to based on the access method used (e.g., if the access method is DPH, it typically would produce an object variable binding; conversely if the access method is RPH, it would typically produce a subject variable binding).  The cost of producing these bindings is estimated based on the average number of outgoing (DPH) or incoming (RPH) edges in most cases, unless the triple happens to have a popular node which appears in a *top K* set.  Other complex nodes such as EXISTs, UNION or OPTIONAL nodes evaluate their costs recursively by planning for their subtrees. (See https://github.com/Quetzal-RDF/quetzal/tree/master/doc/QuetzalPlanner.pdf)  The planner then chooses the cheapest node to schedule first.  Once it has chosen a node, the set of available variables has changed, so a new of cost computations are performed to find the next step.  The planner proceeds in this manner till there are no more available nodes to plan.  The output of the planner is ``com.ibm.research.rdf.store.sparql11.planner.Plan``, which is basically a binary plan tree that is composed of AND plan nodes, LEFT JOIN nodes, etc.  This serves as the input for the next step.
* SQL generator:  In the current implementation, the plan serves as input to a number of SQL templates, which get created for every type of node in the plan tree.  The ``com.ibm.research.rdf.store.sparql11.sqltemplate`` package contains the templates, which generate SQL modularly per node in the plan tree using common table expressions (CTEs).  The template code is general purpose and keeps track of things such as the specific CTE to node mappings, what external variables need to be projected, which variables should be joined together etc.  The actual job of generating SQL for different backends is accomplished using specialized String Templates from the [String Template](http://www.stringtemplate.org) library.  Example files are ``com.ibm.research.rdf.store.sparql11.sqltemplate.common.stg`` which has the templates that are common to all backends.

For more information on how to get started, click on the Wiki to this repository
