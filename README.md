# ETL-pipelines


## Copy multiple files from ADLS as soon as the Status changed
* Overview: In this demo I would like to showcase how to copy multiple files from the ADLS storage based on the metadata control table status.
Here is the high level architecture diagram of the process:

<div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/af87dedd1d6e199526eb135c71ab24cf0bd2c1f5/etl_adls_df_adls.jpg" title="DF_pipeline" alt="DF_pipeline" width="440" height="440"/>&nbsp;
</div>

* Technology: Azure Data Lake Gen2, Sql Server, Data Factory.
* Output: Ready to use parameterized pipeline to dynamically copy the files from a ADLS folder as soon as the Control Table's status is changed.

