# ETL-pipelines


## Copy multiple files from ADLS as soon as the Status changed
* Overview: In this demo I would like to showcase how to copy multiple files from the ADLS storage based on the metadata control table status.
Here is the high level architecture diagram of the process:

<div>
  <img src="https://github.com/katyakoss/ETL-pipelines/blob/af87dedd1d6e199526eb135c71ab24cf0bd2c1f5/etl_adls_df_adls.jpg" title="DF_pipeline" alt="DF_pipeline" width="540" height="440"/>&nbsp;
</div>

* Technology: Azure Data Lake Gen2, Sql Server, Data Factory.
* Output: Ready to use parameterized pipeline to dynamically copy the files from a ADLS folder as soon as the Control Table's status is changed.
* Requirement:
To copy the parquet files from input folder in the ADLS storage account to the output folder in the ADLS storage account based on the status of the file in the metadata control table. If the Status is 'Y' the corresponding table has to be copied.


# Pipeline:
* Firstly, to retrieve the list of all the files we have in our storage account we need to use the "Get Metadata" pipeline activity, which will return the list of file names as an array. Make sure to set up the Field List argument as Child Items to get the array in the ChildItems property.

Once we have the array of the file names, we can use the "For Each" activity to loop through the items of the array:
@activity('Get Metadata1').output.childItems

Next we want to use an "Until" activity to loop through the Flag and exit the loop once the Flag status is changed to 'Y', however nested "Until" activity is not supported inside the "For Each" activity, so I have to use "Execute Pipeline" to be able to use the "Until"

!!Variable Flag

In "Until" activity we need to check our variable 'Flag' and convert it to boolean. This is the expression is @bool(variables('Flag'))

!!Pipeline Parameter

As we have the file name in the main (parent) pipeline, we have to pass it to the inner (child) pipeline. I will do so by firstly parameterising the inner pipeline, and then providing the valuein the Execute pipeline settings:
@item().name

Inside Activities of the "Until" we need to add a "Lookup" activity which will have our SQL database Control Table as a source and will run a query to return only the Flag column for each row, one by one. This is how the query looks like:

select [Flag] from [dbo].[ControlTable] where [File] = '@{pipeline().parameters.File}'


Once we have the status of the particular file, we need to make a decision whether we need to copy it or not. I add the "If Condition" activity and specify the expression as 
@equals(activity('Lookup1').output.firstrow.Flag,'Y')


If it evaluates to True, I add a "Set variable" activity to the True block. "Set variable" activity will set the Flag pipeline variable to True.
I will skip the False block.

Now I am ready with the "Until" activity block. I checked the condition and set up the variable to True if it's Flag value has changed to 'Y' which will exit the Until loop. 
The next and last step is to set up the 'Copy' activity to copy the data from input to output folder. The goal is to create a parameterized datasets in the 'Copy' activity which will be used as a source and as a sink.
To do it dynamically I need to create a parameter on the dataset level and then use this parameter to specify the path to the dataset:

Returning back to the Copy activity, in the Source tab the parameter value has to be added as follows:
@pipeline().parameters.File

I complete the same steps for the Sink tab of the Copy activity. Adding a parameter to use in the folder path.

At this point I have the complete pipeline. Once any of the files flag will be changed to 'Y', the Util loop will evaluate to True and will copy the corresponding File to the output folder.