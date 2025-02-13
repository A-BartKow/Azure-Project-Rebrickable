# Azure-Project-Rebrickable
## Purpose and scope of project

This project bases on the data from [Rebrickable](https://rebrickable.com) web page. Rebrickable is the portal which keeps the registry of all sets, minifigs, parts and all other entites of LEGO in one online catalog.
The purpose of this project was to build the first very simple Data warehouse with Fact table and few Dimensions and going through all the stages of data engineering including:
1. Ingest phase
2. Transform phase
3. Serve phase

The solution is implemented with use of Azure Cloud Data services. 
All Azure Services with possibility to configure repository has been configured with GitHub repositories.

> :memo: **Note:** This project is purely educational and was used to familiarize with and learn different Azure Services used in Data engineering.
> Some of the solutions applied do not fit for the requirements e.g. the size of data warehouse tables and indexes configured against them.
> Any deviation from best practises have been explained and usually formed with some 'assumption' to show the understanding of the concept in theory.

## Solution explanation
### Ingest phase
#### Data source

The are two different data sources which have been used in this project:
1. .csv files containing all master data for entities - further referred as `LEGO` data source
2. .json files recieved from calling Rebrickable API containing user's specific LEGO collections - further referred as `Users` data source

#### Azure services created for this phase

Following Azure services have been created for ingestion phase:
![image](https://github.com/user-attachments/assets/f217b033-db15-468a-b759-95f555383b62)
- Azure Storage Account - `rebrickabledevadlsgen2` - with hierachical structure enabled and LRS type of Redundancy
- Azure Data Factory - `rebrickabledevadf` - the respository for ADF is avaliable in /data-factory/ folder in this repository
- Azure Key Vault - `rebrickabledevakv` - with permission model RBAC enabled to keep all the secrets and passwords in secure place

##### Azure Data Factory content

Following Azure Linked Services has been created in Azure Data Factory:
![image](https://github.com/user-attachments/assets/3a95600a-99a8-4684-8639-634b57067b7f)

##### Azure Data Factory logic

To make solution the most configurable and flexible in terms of adding the news datasets to ingest without doing the changes in pipeline the Configuration table `RebrickableConfig` has been created for adding/removing the dataset for ingestion phase:
![image](https://github.com/user-attachments/assets/2fa4e3e2-3909-4ab2-951f-4c3a4a696f6c)
- `SourceName` - distinguish between two different formats of data
- `DatasetName` - the name of entity from Rebrickable model
- `FileName` - name of the file to which the data is saved on Azure Blob Storage/read from Web page or name of the entity which is used in API call to get the entity data

##### Azure Data Lake Storage structure

The following structure for data ingestion has been designed:
- raw - container which is going to keep the raw, unzipped data as is from source
- Rebrickable - name of the project
- [SourceName] - either 'Lego' or 'Users'
- [DatasetName] - lists of datasets is defined in `RebrickableConfig` table
- year/month/day - date of ingestion generated dynamically during pipeline execution
- filename with extension


#### Security controls for ingestion phase

1. Azure Key Vault GET method is used to retirive the secrets/tokens from Key Vault and the content is hidden behind enabled Secure output/input function on Web activity.
2. All the Linked Services have been created with Managed Identity authentication type
3. Following MS Entra groups have been created with minimum least privilege to grant the proper permissions for MIs:
![image](https://github.com/user-attachments/assets/75963869-7423-4ec6-a23a-90e095403696)
4. Azure Logic App has been created for sending an email to Gmail provider when copy of any of the datasets fails
![image](https://github.com/user-attachments/assets/170e9a32-1c2e-4ffd-a5f4-c221d9ded2be)
![image](https://github.com/user-attachments/assets/c0273de8-80a1-4050-bb0b-96bfde95f7c1)

![image](https://github.com/user-attachments/assets/35a7d4fb-bde1-4816-add3-53f607d5ddd4)


#### Additional features

Lifecycle management policies has been defined and enabled for Azure Blob Storage

#### CI/CD process for Azure Data Factory with use of Azure DevOps pipelines
The CI/CD process for Azure Data Factory has been configured with the use of ARM templates.
The whole process has been designed as described in [Rebrickable-CI-CD readme file](https://github.com/A-BartKow/Rebrickable-CI-CD/blob/main/data-factory/devops-build-n-deploy-stages/readme.md) which was forked from [Azure-enteprise-templates](https://github.com/MarczakIO/azure-enterprise-templates) repository.

The deployment was done on development and production environment, after initial build of templates.

The yaml files created for this project are in /devops/ directory of this repository.

## Transform phase

The aim of the transform phase was to developed the Star schema small datamart solution. The data mart is built with below tables:

1. Date - dimension table with dates
2. Profile - dimension table with registered user's profile id
3. Sets - dimension table with all the avaliable lego sets data in Rebrickable database
4. Owned Sets - fact table containing all sets which are in particular's users collection with the date when the set was acquire by user

#### Azure services created for this phase

Following Azure services have been created for transform phase:
![image](https://github.com/user-attachments/assets/d2d7701e-a68f-4872-b13b-0119e436dea7)
- Azure Databricks Service workspace - `rebrickabledevdatabricks` - as a Trial 14 days free pricing tier
- Azure Key Vault  - `rebrickabledevakvap` - with Vault access policy permission model

##### Azure Databricks cluster

The cluster has been created in way to manage the costs for this educational project the most effectively.
The cluster was created with:
- access mode as `Single user`, 
- without `Photon Acceleration`,
- node type `Standard_DS3_v2`
- terminate after 10 minutes of activity

![image](https://github.com/user-attachments/assets/7a67ac94-b406-496d-8748-5ba7bb368ce6)


##### Azure Databricks authentication with ADLS Gen2

To be able to connect to `Rebrickabledevadlsgen2` ADLS the Secret Scope + Service Principal method was implemented.
Since already created Azure Key Vault `rebrickabledevakv` has the RBAC permission model, the new AKV was created with `Vault access policy` model to be able to implement chosen authentication type.
1. Service Principal for Databricks `Rebrikcable-Dev-Databricks` has been created and added to Storage Blob Data Contributor Entra group:
![image](https://github.com/user-attachments/assets/1c1673ec-1da2-4324-bd89-d76ac714b1b5)
2. Databricks Service Principal `Databricks-Dev-Rebrickable` secret has been placed in `rebrickabledevakvap` Azure Key Vault.
![image](https://github.com/user-attachments/assets/82de51b2-fcf5-402f-ad9c-81fed2d4cf04) 
3. Databricks Service #secrets/createScope was used and and provided the properities for `rebrickabledevakvap` Azure Key Vault.
![image](https://github.com/user-attachments/assets/53e76de9-27d2-4698-9aa7-66d1064242a1)

##### Azure Databricks external table registration

Folowing configuration has been done to set up the external table in `cleansed` container:
1. Access Connector for Azure Databricks `Rebrickabledevadbconnector` created in Azure Portal with Managed Identity option set as On.
   ![image](https://github.com/user-attachments/assets/733f3fb3-7049-4ffd-8dce-dc0aa0c1c045)
2. Created new Storage Credential in Databricks Unity Catalog with Access Connector `Rebrickabledevadbconnector` Managed Identity
   ![image](https://github.com/user-attachments/assets/1151f102-8b02-4ffb-9d1e-6ced2d00b57d)
3. Created an external location for cleansed container.
   ![image](https://github.com/user-attachments/assets/b91d045e-cc0c-40c2-8385-1244454d5def)

##### Azure Data Lake Storage container structure and transform phase logic

The following strucutre of containers has been designed for transforming activities:
1. `curated` - data taken from raw container and saved as delta format without any transformations
2. `cleansed` - data taken from curated container after cleaning and transformation saved as external table as a dataset named folder without any partitioning
![image](https://github.com/user-attachments/assets/90524b2f-1745-49a0-bbff-415166b17bdd)


The following notebooks have been developed and used in transformation phase 
(all notebooks are available in [Azure Rebrickable Project Databricks](https://github.com/A-BartKow/Azure-Rebrickable-Project-Databricks) repository with detailed explanation of content)
1. `Rebrickable - design and full load` - notebook for doing the initial load to curated container, design and register the external tables for Dimensions and Facts tables
2. `Rebrickable - incremental load` - notebook used for daily load of new data - saving new files as delta in curated container and updating the external tables with new data

## Serve Phase

The aim of that phase was to load the external data tables located on `rebrickabledevadlsgen2` as simple SQL tables stored in SQL dedicated pool for allowing the end user access it from SQL Server Management Studio software.

### Azure services created for this phase

1. Azure Synapse Analytics workspace - `rebrickabledevsynapse`
2. Dedicated SQL pool - `Rebrickabledevsqldw` - dedicated sql pool created in `rebrickabledevsynapse` Azure Synapase Analytics services to which the data will be loaded
![image](https://github.com/user-attachments/assets/1afa6817-ec10-421e-ba0f-25dc353c37c6)

#### Copying data from ADLS Gen2 to Dedicated SQL Pool

Within the different various methods for copying the data to SQL dedicated pool Azure Databricks notebook with `spark.write.format('sqldw')` method was used.
The reason for using that method is the data written as delta format in ADLS Gen 2 `cleasned@rebrickabledevadlsgen2` container which will be used as a source.

Copy methods as PolyBase, COPY INTO or ADF/Synapse pipeline with Copy activity are not avaliable for copying the delta format at the moment.

The Databricks notebook `Rebrickable - loading data to Dedicated SQL Pool` with logic for copying the data to SQL dedicated pool 'Rebrickabledevsqldw` is avaliable in [Azure Rebrickable Project Databricks](https://github.com/A-BartKow/Azure-Rebrickable-Project-Databricks) repository.

#### Dedicated SQL pool design

The tables equivalent to external tables registered on `rebrickabledevadlsgen2` have been created in `Rebrickabledevsqldw`:
1. Date - dimension with heap index and replicate distribution
2. Profile - dimension table with heap index and replicate distribution
3. Sets - dimension table with heap index and replicate distribution
4. Owned Sets - fact table with clustered columnstore index and hash distribution on users' profile id

The data from `cleansed` container on `rebrickabledevadlsgen2` is initially loaded to `Rebrickabledevsqldw.dbo` temporary schema. Later on it is being loaded to `Rebrickabledevsqldw.Rebrickable` schema as part of SQL script.

The whole SQL script for designing and loading the data (including some additional features set up as column-level and row-leve security) to `Rebrickabledevsqldw.Rebrickable` schema is published in /synapse/sqlscript/Rebrickable-SLQ DW design.json file in this repository.

### Querying data in SQL Server Management Studio

Following screenshot is showing the database content of `Rebrickabledevsqldw` which is avaliable for querying the data:
![image](https://github.com/user-attachments/assets/7c5ed73e-69ea-479d-a5b4-0e413f64655b)

![image](https://github.com/user-attachments/assets/ce973899-bdb7-4b84-9b64-3f28c72bdb10)

## Azure Data Factory orchestration for daily loads

Following pipeline has been created to get the daily loads of data and inserting/upserting it to `Rebrickabledevsqldw.dbo` schema
![image](https://github.com/user-attachments/assets/5dcdb316-e5a2-4345-8466-aa531860ec75)

The `Execute Rebrickable - Incremental load` and `Execute Rebrickable - loading data to SQL Pool` activities have the job cluster creation option enabled to create, execute and deprovision the cluster after the referenced notebook's execution is completed.


## To Dos and Next Steps for Rebrickable project

There still some improvements for Rebrickable project which I think of to introduce:
1. Script the Production environment creation by using Terraform
2. Automate 'Serve phase' part of SQL data loading from `Rebrickabledevsqldw.dbo` schema to `Rebrickabledevsqldw.Rebrickable` as part of daily/incremental load (the `Rebrickable-SLQ DW design` SQL script was executed only once and is not configured for incremental loads)
3. Create a proper monitoring using Azure Monitor cloud service.



