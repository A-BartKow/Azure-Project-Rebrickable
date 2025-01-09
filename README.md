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
### Data source
The are two different data sources which have been used in this project:
1. .csv files containing all master data for entities - further referred as `LEGO` data source
2. .json files recieved from calling Rebrickable API containing user's specific LEGO collections - further referred as `Users` data source.

### Azure services used in this phase
Following Azure services has been created for ingestion phase:
![image](https://github.com/user-attachments/assets/f217b033-db15-468a-b759-95f555383b62)
- Azure Storage Account - `rebrickabledevadlsgen2` - with hierachical structure enabled and LRS type of Redundancy
- Azure Data Factory - `rebrickabledevadf` - the respository for ADF is avaliable in /data-factory/ folder in this repository
- Azure Key Vault - `rebrickabledevakv` - with permission model RBAC enabled to keep all the secrets and passwords in secure place

#### Azure Data Factory content
Following Azure Linked Services has been created in Azure Data Factory:
![image](https://github.com/user-attachments/assets/3a95600a-99a8-4684-8639-634b57067b7f)

#### Azure Data Factory logic
To make solution the most configurable and flexible in terms of adding the news datasets to ingest without doing the changes in pipeline the Configuration table `RebrickableConfig` has been created for adding/removing the dataset for ingestion phase:
![image](https://github.com/user-attachments/assets/2fa4e3e2-3909-4ab2-951f-4c3a4a696f6c)
- `SourceName` - distinguish between two different formats of data
- `DatasetName` - the name of entity from Rebrickable model
- `FileName` - name of the file to which the data is saved on Azure Blob Storage/read from Web page or name of the entity which is used in API call to get the entity data

#### Azure Data Lake Storage structure
The following structure for data ingestion has been designed:
- raw - container which is going to keep the raw, unzipped data as is from source
- Rebrickable - name of the project
- [SourceName] - either 'Lego' or 'Users'
- [DatasetName] - lists of datasets is defined in `RebrickableConfig` table
- year/month/day - date of ingestion generated dynamically during pipeline execution
- filename with extension


### Security controls for ingestion phase
1. Azure Key Vault GET method is used to retirive the secrets/tokens from Key Vault and the content is hidden behind enabled Secure output/input function on Web activity.
2. All the Linked Services have been created with Managed Identity authentication type
3. Following MS Entra groups have been created with minimum least privilege to grant the proper permissions for MIs:
![image](https://github.com/user-attachments/assets/75963869-7423-4ec6-a23a-90e095403696)
4. Azure Logic App has been created for sending an email to Gmail provider when copy of any of the datasets fails

### Additional features
Lifecycle management policies has been defined and enabled for Azure Blob Storage



