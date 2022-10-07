# Run the community detection algorithm

## Introduction

In this lab, using the graphs which are created in the previous labs, we applied the community detection algorithm which identifies the communities within the graphs. The community detection algorithm takes the input graphs and identities strong connectivity within graphs and forms multiple smaller communities. Infomap is used for community detection in this lab.

Estimated Lab Time: 15 minutes

### Objectives

In this lab, you will:

* Create a graph in Graph Studio.
* Detect the communities using the Infomap.

### Prerequisites

This lab assumes you have:

* All previous labs were completed successfully.

## Task 1: Create the Bucket in OCI Compartment

1. Clone the DRA Community Detection Client from the github.

git clone <https://github.com/oracle/DRACommunityDetection.git>

1. Update the db-config.properties file.

  Update the value for the below properties.

   tenant   - tenant OCID
   database - Name of the Database
 username - Username to login to database
 password - Password to login to database
 endpoint - Endpoint for connecting to Autonomous Database instance

1. Update the graph-config.properties file.

 Update the value for the below properties.

   graph_name - Name of the graph created in Graph Studio.
   vertex_property_column
 edge_property_source_column
 edge_property_destination_column
 edge_property_weight_column

## Task 2: Compile and Run the Community Detection

1. Compile the maven project

 mvn compile

2.Execute the project to see the identified clusters using the Infomap Algorithm

 mvn exec:java -Dexec.mainClass=com.oracle.ms.app.InfomapGraphClient 'MaxNumberOfIterations'

 Where

* com.oracle.ms.app.InfomapGraphClient - Main class which loads the graph and runs the Infomap to identify the Clusters.

Output:

## Learn More

## Acknowledgements

* **Author** - Praveen Hiremath, Developer Advocate
* **Contributors** -  Praveen Hiremath, Developer Advocate
* **Last Updated By/Date** - Praveen Hiremath, Developer Advocate, September 2022
