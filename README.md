# ETL-pipelines


## Copy multiple files from ADLS as soon as the Status changed
* Overview: In this demo I would like to showcase how to copy multiple files from the ADLS storage based on the metadata control table status.
Here is the high level architecture diagram of the process:

<div>
  <p style="text-align:center;">
    <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_adls.jpg" title="DF_pipeline" alt="DF_pipeline" width="500" height="400"/>&nbsp;
   </p> 
</div>

* Requirement:
To copy the parquet files from input folder in the ADLS storage account to the output folder in the ADLS storage account based on the status of the file in the metadata control table. If the Status is 'Y' the corresponding table has to be copied.
<div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_adls_input.jpg" title="ADLS_input" alt="ADLS_input" width="240" height="180"/>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_adls_sql_input.jpg" title="SQL_input" alt="SQL_input" width="240" height="180"/>&nbsp;
</div>

* Technology: Azure Data Lake Gen2, Sql Server, Data Factory.
* Output: Ready to use parameterized pipeline to dynamically copy the files from a ADLS folder as soon as the Control Table's status is changed.

# Pipeline:
Firstly, to retrieve the list of all the files we have in our storage account we need to use the "Get Metadata" pipeline activity, which will return the list of file names as an array. Make sure to set up the Field List argument as "Child Items" to get the array in the ChildItems property.

<div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_adls_getMetadata_settings.jpg" title="GetMetadata_settings" alt="GetMetadata_settings" width="440" height="300"/>&nbsp;
</div>

If I debug right now, I see the array of the file names, therefore I can use the "For Each" activity to loop through the items of the array:

<em> @activity('Get Metadata1').output.childItems </em>

<div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_adls_getMetadata_output.jpg" title="GetMetadata_output" alt="GetMetadata_output" width="240" height="500"/>&nbsp;

  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_adls_ForEach_settings.jpg" title="ForEach_Settings" alt="ForEach_Settings" width="240" height="200"/>&nbsp;
</div>

Next I want to use an "Until" activity to loop through the Flag and exit the loop once the Flag status is changed to 'Y', however nested "Until" activity is not supported inside the "For Each" activity, so I have to use "Execute Pipeline" to be able to use the "Until". I am going to create a new pipeline in the Invoked pipeline:

<div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_execute_pipeline_settings.jpg" title="ExecutePipeline_settings" alt="GetMetadata_settings" width="500" height="400"/>&nbsp;
</div>

In the inner pipeline I am creating a variable called Flag to use as an exit from Until loop:

<div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_execute_pipeline_variable.jpg" title="Variable" alt="Variable" width="500" height="400"/>&nbsp;
</div>

I am ready to create the "Until" activity. In it I have to check the variable 'Flag' and to convert it to boolean. This is the expression:
 
 <i> @bool(variables('Flag')) </i>

 <div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_execute_pipeline_Untill_settings.jpg" title="Until" alt="Until" width="500" height="500"/>&nbsp;
</div>

As we have the file name in the main (parent) pipeline, we have to pass it to the inner (child) pipeline. I will do so by firstly parameterising the inner pipeline, and then providing the value in the Execute pipeline settings:

<i> @item().name </i>

 <div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_execute_pipeline_parameter.jpg" title="InnerParameter" alt="InnerParameter" width="500" height="400"/>&nbsp;
</div>

Inside Activities of the "Until" we need to add a "Lookup" activity which will have our SQL database Control Table as a source and will run a query to return only the Flag column for each row, one by one. This is how the query looks like:

<i> select [Flag] from [dbo].[ControlTable] where [File] = '@{pipeline().parameters.File}' </i>

 <div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_execute_pipeline_lookupquery.jpg" title="LookupStatement" alt="LookupStatement" width="400" height="300"/>&nbsp;
</div>

Once I have the status of the particular file, I have to make a decision whether I need to copy it or not. I add the "If Condition" activity and specify the expression as: 

<em> @equals(activity('Lookup1').output.firstrow.Flag,'Y') </em>

 <div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_execute_pipeline_lookup_if.jpg" title="IfStatement" alt="IfStatement" width="500" height="300"/>&nbsp;
</div>

If it evaluates to True, I add a "Set variable" activity to the True block. "Set variable" activity will set the Flag pipeline variable to True.
I will skip the False block.

Now I am ready with the "Until" activity block. I checked the condition and set up the variable to True if it's Flag value has changed to 'Y' which will exit the Until loop. 
The next and last step is to set up the 'Copy' activity to copy the data from input to output folder. The goal is to create a parameterized datasets in the 'Copy' activity which will be used as a source and as a sink.
To do it dynamically I need to create a parameter on the dataset level and then use this parameter to specify the path to the dataset:

 <div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_execute_pipeline_ds-parameter.jpg" title="DsParameter" alt="DsParameter" width="500" height="250"/>&nbsp;
</div>

Returning back to the Copy activity, in the Source tab the parameter value has to be added as follows:
<em> @pipeline().parameters.File </em>

I complete the same steps for the Sink tab of the Copy activity. Adding a parameter to use in the folder path.

 <div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/main/img/etl_adls_df_execute_pipeline_copyactivity_sink.jpg" title="CopyActivity" alt="CopyActivity" width="500" height="400"/>&nbsp;
</div>



At this point I have the complete pipeline. Once any of the files flag will be changed to 'Y', the Util loop will evaluate to True and will copy the corresponding File to the output folder.