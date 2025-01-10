# Azure-Project-Rebrickable
## Purpose and scope of project
This project bases on the data from [Rebrickable](https://rebrickable.com) web page. Rebrickable is the portal which keeps the registry of all sets, minifigs, parts and all other entites of LEGO in one online catalog.
The purpose of this project was to build the first very simple Data warehouse with Fact table and few Dimensions and going through all the stages of data engineering including:
1. Ingest phase
2. Transform phase
3. Serve phase
The solution is implemented with use of Azure Cloud Data services.

## Solution explanation
### Ingest phase
#### Data source
The are two different data sources which have been used in this project:
1. .csv files containing all master data for entities - further referred as `LEGO` data source
2. .json files recieved from calling Rebrickable API containing user's specific LEGO collections - further referred as `Users` data source.

#### Azure services used in this phase
Following Azure services has been created for ingestion phase:
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

#### Additional features
Lifecycle management policies has been defined and enabled for Azure Blob Storage

## Transform phase
The aim of the transform phase was to developed the Star schema small Data Mart solution. The data mart is build with below tables:

1. Date - dimension table with dates
2. Profile - dimension table with registered user's profile id
3. Sets - dimension table with all the avaliable lego sets data in Rebrickable database
4. Owned Sets - fact table containing all sets which are in particular's users collection with the date when the set was acquire by user

#### Azure services used in this phase
Following Azure services has been created for ingestion phase:
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

##### Azure Databricks authentication with ADLS Gen2
To be able to connect to Rebrickabledevadlsgen2 ADLS the Secret Scope + Service Principal method was implemented.
Since already created Azure Key Vault `rebrickabledevakv` has the RBAC permission model, the new AKV was created with `Vault access policy` model to be able to implement chosen authentication type.
1. Service Principal for Databricks `Rebrikcable-Dev-Databricks` has been created and added to Storage Blob Data Contributor Entra group:
![image](https://github.com/user-attachments/assets/1c1673ec-1da2-4324-bd89-d76ac714b1b5)
2. Databricks Service Principal `Databricks-Dev-Rebrickable` secret has been placed in `rebrickabledevakvap` Azure Key Vault.
3. Databricks Service #secrets/createScope was used and and provided the properities for `rebrickabledevakvap` Azure Key Vault.

##### Azure Databricks external table registration
Folowing configuration has been done to set up the external table in `cleansed` container:
1. Access Connector for Azure Databricks created in Azure Portal with Managed Identity option set as On.
2. Created new Storage Credential in Databricks Unity Catalog with Access Connector Managed Identity
3. Created an external location for cleansed container.

##### Azure Data Lake Storage container structure and transform phase logic
The following strucutre of containers has been designed for transforming activities:
1. `curated` - data taken from raw container and saved as delta format without any transformations
2. `cleansed` - data taken from curated container after cleaning and transformation saved as external table as a dataset named folder without any partitioning

The following notebooks have been developed and used in transformation phase 
(all notebooks are available in [Azure Rebrickable Project Databricks](https://github.com/A-BartKow/Azure-Rebrickable-Project-Databricks) repository)
1. `Rebrickable - design and full load` - notebook for doing the initial load to curated container, design and register the external tables for Dimensions and Facts tables
2. `Rebrickable - incremental load` - notebook used for daily load of new data - saving new files as delta in curated container and updating the external tables with new data







##### Azure Data Factory orchestration for daily loads







