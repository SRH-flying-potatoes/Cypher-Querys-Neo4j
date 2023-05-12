# Cypher-Querys-Neo4j
**Cypher Query used to create data base ,graph projections and applying Dijkstra's Algorithm.**

**// STEP 1: Creating uniqueness constraints for Airport, City, and Country labels**

CREATE CONSTRAINT airports_unique IF NOT EXISTS FOR (a:Airport) REQUIRE a.iata IS UNIQUE;

CREATE CONSTRAINT cities_unique IF NOT EXISTS FOR (c:City) REQUIRE c.name IS UNIQUE;

CREATE CONSTRAINT countries_unique IF NOT EXISTS FOR (d:Country) REQUIRE d.code IS UNIQUE;


**// STEP 2: Loading Airport data from CSV file and creating the Airport Data graph**

LOAD CSV WITH HEADERS FROM 'file:///path/to/airport_data.csv' AS row

MERGE (airport:Airport {iata: row.iata})

MERGE (ci:City {name: row.city})

MERGE (co:Country {code: row.country})

MERGE (a)-[:IN_CITY]->(ci)

MERGE (a)-[:IN_COUNTRY]->(co)

MERGE (ci)-[:IN_COUNTRY]->(co)

SET a.id = row.id,

    a.icao = row.icao,
    
    a.city = row.city,
    
    a.descr = row.name;

**// STEP 3: Loading Airport routes data from CSV file and creating the Airport Routes graph**

LOAD CSV WITH HEADERS FROM 'file:///path/to/airport_routes.csv' AS row

MATCH (source:Airport {iata: row.source})

MATCH (target:Airport {iata: row.destination})

MERGE (source)-[route:HAS_ROUTE]->(target)
ON CREATE SET 

    route.distance = toInteger(row.distance);

**// STEP 4: Projecting the graph for shortest path calculation**

CALL gds.graph.project(

    'Flying-Potatoes',
    
    'Airport',
    
    'HAS_ROUTE',
    
    {
    
        relationshipProperties: 'distance'
        
    }
    
) YIELD

  graphName, nodeProjection, nodeCount, relationshipProjection, relationshipCount;

**// STEP 5: Finding the shortest path using Dijkstra algorithm**

MATCH (source:Airport {iata: '000'}), (target:Airport {iata: '000'})

CALL gds.shortestPath.dijkstra.stream('Flying-Potatoes', {

    sourceNode: source,
    
    targetNode: target,
    
    relationshipWeightProperty: 'distance'
    
})

YIELD index, sourceNode, targetNode, totalCost, nodeIds, costs, path

RETURN

    index,
    
    gds.util.asNode(sourceNode).iata AS sourceNodeName,
   
    gds.util.asNode(targetNode).iata AS targetNodeName,
    
    totalCost,
    
    [nodeId IN nodeIds | gds.util.asNode(nodeId).iata] AS nodeNames,
    
    costs,
    
    nodes(path) AS path
    
ORDER BY index;

