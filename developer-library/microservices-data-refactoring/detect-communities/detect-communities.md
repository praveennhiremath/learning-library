# Run the community detection algorithm

## Introduction

In this lab, using the graphs which are created in the previous labs, we applied the community detection algorithm which identifies the communities within the graphs. The community detection algorithm takes the input graphs and identities strong connectivity within graphs and forms multiple smaller communities. Each smaller communities are the reffered to schemas. Overall, It decomposes the monolith database schema(graph) into modular microservice schemas(smaller communities). Here we used Oracle Graphs for representations and the usage of the Louvain Method via the APIs exposed by the PGX library. Louvain method is used for community detection in this lab.

Estimated Lab Time: 15 minutes

### Objectives

In this lab, you will:
* create a graph in Data Studio
* Detect the communities using the Infomap.

### Prerequisites (Optional)


This lab assumes you have:
* An Oracle account.
* All previous labs were completed successfully.

## Task 1: Create the Bucket in OCI Compartment.
	
1. Login to OCI > Storage > Object Storage & Archive Storage > Buckets

	![Image alt text](./images/sample1.png)

2. Select the compartment and Click on Create Bucket
	
	![Image alt text](./images/create-bucket-click.png)
	
3. Enter Bucket Name and Click Create
	
	![Image alt text](./images/enter-bucket-name.png)	
	

## Task 2: Upload the CSV files of NODES and EDGES in OCI Buckets

1. Select the bucket and click on upload

  ![Image alt text](./images/click-on-upload.png)
  
2. Drag and Drop the Nodes and Edges files and click on Upload

  ![Image alt text](./images/drag-and-drop-upload.png)

2. View the main graph - Delete this
	~~~<copy>
	%pgql-pgx
	select * from match (s)-[t]->(d) on cerner_graph 
	</copy>~~~


## Task 3: Load the Graph Configuration

1. Create a file with below content and import it in graph configuration

	- 
	{
    "format": "csv",
    "vertex_props": [
        {
            "name": "TABLE_NAME",
            "type": "string"
        }
    ],
    "edge_props": [
        {
            "name": "TOTAL_AFFINITY",
            "type": "double"
        }
    ],
    "vertex_id_strategy": "keys_as_ids",
    "edge_id_strategy": "keys_as_ids",
    "vertex_id_type": "string",
    "partition_while_loading": "no",
    "loading": {
        "create_edge_id_index": true,
        "create_edge_id_mapping": true
    },
	
    "vertex_uris": [
        "https://objectstorage.us-ashburn-1.oraclecloud.com/n/maacloud/b/bucket-medical-recs-csv/o/NODES_259_NEW.csv"
    ],
    "edge_uris": [
        "https://objectstorage.us-ashburn-1.oraclecloud.com/n/maacloud/b/bucket-medical-recs-csv/o/EDGES_259_NEW.csv"
    ],
    "vertex_id_column": "TABLE_NAME",
    "edge_id_column": "TABLE_MAP_ID",
    "edge_source_column": "TABLE1",
    "edge_destination_column": "TABLE2",
    "header": true

	}
	Replace values for "vertex_uris" and "edge_uris" with newly uploaded csv files.
	Save this as MEDICAL_REC_GRAPH.json file. 
	upload this json file in graph configurations.
	
## Task 3: Detect the communities using Infomap

1. create the new Notebook
	Create new paragraph and add below content and run the paragraph
	~~~<copy>
	%pgx-java
	PgxGraph graph = session.readGraphWithProperties(MEDICAL_REC_GRAPH, "MEDICAL_REC_GRAPH");
	</copy>~~~

3. Visualize the graph 

	~~~<copy>
	%pgql
	select * from match (s)-[t]->(d) on MEDICAL_REC_GRAPH
	</copy>~~~
3. Run the Infomap to find the communities within the graphs
	~~~<copy>
	

	%pgx-java
	EdgeProperty<Double> weight = graph.getEdgeProperty("TOTAL_AFFINITY");
	VertexProperty<Integer, Double> rank = analyst.weightedPagerank(graph, 1e-16, 0.85, 1000, true, weight);
	VertexProperty<Integer, Long> module = graph.createVertexProperty(PropertyType.LONG);

	Partition<Integer> promise = analyst.communitiesInfomap(graph, rank, weight, 0.15, 0.0001, 20, module);
	VertexCollection<Integer> first_component = promise.getPartitionByIndex(0)

	for (VertexCollection<Integer> partition : promise){
		out.println("==================================");
		for (PgxVertex<Integer> vertexInCommunity : partition){
			out.println(vertexInCommunity.getId());
		}
		out.println("No of Nodes in Cluster : " + partition.size());
		
	}
	</copy>~~~

## Learn More

## Acknowledgements
* **Author** - Praveen Hiremath, Developer Advocate
* **Contributors** -  Praveen Hiremath, Developer Advocate
* **Last Updated By/Date** - Praveen Hiremath, Developer Advocate, September 2022 

