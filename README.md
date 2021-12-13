# Event Notified ETL Template App

## Overview

Event driven architecture is becoming increasingly common in both transactional and analytical processing. According to Martin Fowler, an [Event Driven](https://martinfowler.com/articles/201701-event-driven.html) approach can be typically classified into following:

1. ### Event Notification

Events carry only minimum details, just enough to inform the clients about a state change. Clients need to contact source system to get the new state

2. ### Event-Carried State Transfer

Events carry the fully copy of the new state. Clients enough information to do their processing from the event itself.

3. ### Event-Sourcing

Events are recorded in an event store which becomes the principal source of the truth and the system state is derived from it.

4. ### CQRS

Seperate data models are used for reading and writing information. This can be useful where is single model is too complex for reading and writing data.

From data processing or ETL perspective two patterns that are of primary interest are _Event Notification_ and _Event-Carried State Transfer_. Typical stream processing applications such as IOT, real-time telemetry systems usually uses the second approach i.e each event carry a full state. This is quite good if :

1. Fully state can fit in the event pay load, and it can be represented in a structured format

2. Occurs at regular intervals and have high frequency.

3. Event sources can be classified in to small number of categories.

However what if the state change is sporadic, content is unstructured or large , and sources are quite varied. For example lets say you are building a system to perform analytics on large types of financial statements or forms. Some of these files e.g. General Ledger might be very large and might not have the same schema. _Event-Carried State Transfer_ might not be the right choice here and traditional batch process based on regular schedule ETL, polling for state changes and long orchestrated data pipelines might be awkward at best and inefficient and highly coupled at worst.

Event notified ETL is an excellent alternative in such cases. It has the following key benefits

1. Low level of coupling between data sources, data processes and consumers, which allows for higher evolvability. Allows you to introduce new sources, consumers and data processing logic without any significant changes to the existing.

2. Events become natural interfaces and helps break up the long ETL pipelines into technical and functional boundaries. This helps to reason, build and explain individual pipeline segments with good alignment to these boundaries and reduces the complexity and difficulty in maintaining long chain of ETL pipelines typically associated with batch systems.

This repo contains a sample application which is build using the _Event Notification_ pattern. It uses NYC Taxi data and is build using Azure Data Platform components.

![Design](https://github.com/bablulawrence/event-notified-etl/blob/main/design/app-design.svg)

## Application Components

### Sharepoint

Two sharepoint sites are used for managing file ingestion.

### Logic Apps

Logic apps are used for event processing.

### Databricks Jobs and Notebooks

Azure Databricks Jobs and Notebooks are used for data processing.

### Azure Data Lake Storage Gen2 and Delta Tables

Azure Data Lake Storage Gen2 and Delta Tables are used for data storage.

## Application Deployment

### Deploy Azure Resources

#### 1. Fork this repo

Fork this repo by clicking the _fork_ button on the top-right of the repository page.

#### 2. Create an Azure Resource Group

Create a resource group for holding Azure resources required for the app. You can do this by running below Azure CLI command in Azure Cloud Shell.

`az group create --location {location} --name {resource group name}`

example :

`az group create --location southeastasia --name rg-event-etl`

#### 3. Create a Service Principal for Azure deployment

Run below command to create a Service Principal scoped to the resource group created in [step 2](#2-create-an-azure-resource-group).

```azurecli
az ad sp create-for-rbac --name "EventEtlSP" \
--role contributor \
--scopes /subscriptions/{subscription id}/resourceGroups/{resource group name}
```

example :

```azurecli
az ad sp create-for-rbac --name "EventEtlSP" \
--role contributor \
--scopes /subscriptions/1ee5ed92-933d-4c51-ac9f-96329a4273f7/resourceGroups/rg-event-etl
```

output will be like :

```azurecli
{
    "clientId": "<GUID>",
    "clientSecret": "<GUID>",
    "subscriptionId": "<GUID>",
    "tenantId": "<GUID>",
    (...)
}
```

hold on to this, you will need in the next step.

#### 4. Add Github Secrets

Create the following secrets variables under your repository - _Settings -> Secrets_. The secret names should match exactly what is given below.

| #   | Secret Name       | Description                                                                                                      |
| --- | ----------------- | ---------------------------------------------------------------------------------------------------------------- |
| 1   | AZURE_CREDENTIALS | Service Principal Details for deployment, [output of step 3](#3-create-a-service-principal-for-azure-deployment) |

#### 5. Run Github workflow

Under `Actions` in your forked repository, you will find following workflows. Run them manually by clicking _workflow name -> run workflow_, in the sequence given below.

| #   | Workflow         | Description                                | Sequence  | Triggers |
| --- | ---------------- | ------------------------------------------ | --------- | -------- |
| 1   | Deploy Resources | Provisions application components in Azure | run first | Manual   |

### SharePoint Setup

Create Two SharePont Team Sites - Green Taxi and Yellow Taxi and create following folders to upload taxi data

- Green Taxi - Documents/monthly
- Yellow Taxi - Documents/daily

You will the data to upload in /data folder of this repo

### Power Automate Setup

Login to Power Automate and create connection to SharePoint and Azure Event Grid. Once the connections are setup, import the flows to publish the events from /power_automate of this repo.

### Logic app connection authentication

Login to Azure Portal and authenticate following logic app connections.

| #   | Connection            | Description                                            | Authentication Type                |
| --- | --------------------- | ------------------------------------------------------ | ---------------------------------- |
| 1   | sharepointonline      | Connection to SharePoint read data                     | OAuth 2.0 Authorization Code Grant |
| 2   | azureeventgrid        | Connection consume events from Azure Event Grid        | OAuth 2.0 Authorization Code Grant |
| 3   | azureeventgridpublish | Connection to publish events to Azure Event Grid       | SAS Key                            |
| 4   | azureblob             | Connection to Azure Data Lake Gen2 read and write data | Managed Identity                   |
| 5   | powerbi               | Connection to refresh PowerBI Datasets                 | OAuth 2.0 Authorization Code Grant |

### Publish PowerBI datasets and reports

1. Open the PBIX file in /powerbi repo using Power BI Desktop. Update the following in the connection string

   1. Databricks workspace URI
   2. Cluster Id
   3. Personal Access Token

2. Login to Power BI service and publish the PBIX file to your workspace.

### Mount ADLS Gen2 in Databricks

1. Create an Azure AD app and client secret.
2. Create databricks scope and store the client secret value in it using the following commands

   ```bash
   databricks secrets create-scope --scope datalake
   databricks secrets put --scope datalake --key adappsecret

   ```

3. Open the notebook `nb-01-setup.py` in the Databricks workspace and update the `clientId`, `scope`, `key`, `storageAccountName` and with the values from step 2. Run the notebook to mount the data lake directory in Databricks

## Running the application

1. Upload the Green Taxi data files to the SharePoint site. Following series of events will be raised as the workflows are executed.

   1. `NYCTaxi.GreenTaxi.TripData.FileIngested`
   2. `NYCTaxi.GreenTaxi.TripData.RawFileUploaded`
   3. `NYCTaxi.GreenTaxi.TripData.ProcessRawStarted`
   4. `NYCTaxi.GreenTaxi.TripData.FileCleansed`
   5. `NYCTaxi.GreenTaxi.TripData.ProcessCleansedStarted`
   6. `NYCTaxi.GreenTaxi.TripData.ZoneSummaryCurated`
   7. `NYCTaxi.GreenTaxi.TripData.ZoneSummaryDataRefreshed`

   If any of the steps fail, an error event with suffix `.Failed` e.g. `NYCTaxi.GreenTaxi.TripData.FileCleansed.Failed` will be generated. Similar events will be generated when repeat the process for Yellow Taxi data.

2. Execute the job `jb-04-zone-lookup-process-all`. This will load the NYC Zone Lookup data and create the necessary hive tables for data consumption

3. Run the sample queries in notebook `nb-05-nyc-taxi-validate` to check the data.

4. Login to the Power BI and run the nyc-taxi reports to explore the Total Passenger Count By Borough, Zone and Taxi Type
