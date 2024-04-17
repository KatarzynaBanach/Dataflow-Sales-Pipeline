# Dataflow Sales Pipeline
**_IN PROGRESS_**

### **STACK:**

_python, Apache Beam, GCP, Dataflow, BigQuery, Cloud Storage, Terraform, bash, yaml_

### **RATIONALE:**

Practice and mistakes are the best teacher. Therefore building my own pipeline, facing various challenges and requirements, coming up with 'extra' features and then trying to implement them as well as looking through the GCP documentation with Eureka moments was a great deep-dive into Cloud Data Engineering world.

### **PROJECT RESOURCES:**
![obraz](https://github.com/KatarzynaBanach/Dataflow-Sales-Pipeline/assets/102869680/fa6c674c-e16b-4f00-8f39-fb4b86d46d00)

### **FLOW & FEATURES OF THE SALES PIPELINE:**
* Buckets are created using **Terraform** and raw data is **loaded using cloud shell**.
* Parameters such as **project, region** and type of **runner** are passed **as command line arguments**. 
* **Configuration** and **table schemas** are taken **from .yaml** file.
* Files with rows count and status from the previously run pipeline are **removed from bucket**. 
* **Dataset is created** (in not exists) 
* **Raw data** in form of .**csv and .txt is loaded** from the client's bucket.
* Sales data is cleaned: some **columns are joined** (customer's names), other are split (**the extraction of value and currency** from price), rows with **no id-s are dropped**, deduplication is done.
* Unwanted **symbols are removed** (using regex), **dates are unified**, mapping of sex takes place, **phone numbers** in various formats **are extracted**.
* **System columns**: _created_at and _process_id are created.
* Country codes and names are taken **from different file, split and parsed as dictionary**. Then used as the **side input to replace country code** with a full names.
* All rows and **rows per each brand are counted** and written **to a file in a bucket**.
* **Personal customer data** (such as names and phonenumbers) that is unnecessary to analysis, is loaded **into BigQuery** (since some customers may already have appeared in BigQuery, firstly **data from BigQuery** is taken -> then **old and new data is merged** -> **dedupliacation** takes place -> data is **loaded into BigQuery** with the **override** mode).
* Data is **filtered by brands** (list of brands is taken from .yaml file) and loaded **into separate tables in BigQuery** (with the **append** mode). In case of **a new brand it is enough to add the name to config.file** and during the next run of a pipeline **an new table** will be **automatically** created and filled with data.
* A **status of pipeline** is written **into bucket**.
* Files with **data that have been processed are moved** to another bucket.

### **DATAFLOW JOB:**
![obraz](https://github.com/KatarzynaBanach/Dataflow-Sales-Pipeline/assets/102869680/06c613a4-3058-43c1-bb59-b563a51ac5b2)

### **SET UP:**
Open Cloud Shell and choose project:
```
gcloud config set project <PROJECT_ID>
```
Copy repository:
```
git -C ~ clone https://github.com/KatarzynaBanach/Dataflow-Sales-Pipeline
```
Change directory:
```
cd Dataflow-Sales-Pipeline/Terraform/
```
Change variables in files:
* _main.tf_ (names of buckets - the have to be unique globally)
* _variables.tf_ (if needed)
* _terraform.tfvars_ (project name and path of file with service account keys)
Initialize terraform, check possible changes and apply them::
```
terraform init
terraform plan
terraform apply
```
Come back to _Dataflow-Sales-Pipeline/_ directory.
Change variables in files: 
* _init_settings.sh_ (CLIENT_BUCKET_NAME - to the same name as given in _main.tf_ to bucket _client_data_)
* _config.yaml_ (buckets' names to the same names as given in _main.tf_)
(You can changes also other variables if needed - important it that it the best practice to stick to the same region for all resouces within one project)

Now when all infastructure is set and variables are appropriate the pipeline can be run:
* either directly in cloud shell with proper parameters:
```
python3 sales_pipeline.py --project=<PROJECT_ID> --region=<REGION> <--DirectRunner or --DataflowRunner or empty - default is DirectRunner>
```
* or executing file submit_pipeline.sh with the same command as above, but I find it more convenient (in that case variables in those file need to be changer and proper line of code commented)
```
sh submit_pipeline.sh
```
It takes from 5 to 15 minutes (local running is fasther in case of that pipeline then running it with Dataflow). If --DataflowRunner is choosen you can check job progress and possible error in tab Dataflow -> Jobs.
![obraz](https://github.com/KatarzynaBanach/Dataflow-Sales-Pipeline/assets/102869680/02b134df-e0ef-4379-bda1-53737223822e)
If job status is 'Succeeded' that job was run properly! Congrats!
Details can be found in <output_bucket>/status/pipeline_status.txt and <output_bucket>/status/processed_rows*.txt.
Results will be visible in BigQuery in dataset sales.

**Possible causes of errors:**
* Some required API is not allowed.
* Inconsistencies in variable between terraform, init_setup.py and command line arguments.
* Inconsistencies in regions and zones.
* Lack of appropriate permissions for used service account.
* Resource pool exhausted for Dataflow.
* Lack of resources in a choosen region at the moment -> try another region / zone.
