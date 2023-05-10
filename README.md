# NiFiTaniumToPutDatabaseRecord
NiFi flow that sources from Tanium API results and converts to allow for recordset aware destinations

# Overview
This article describes a NiFi flow that consumes data from Tanium API endpoints. This article can be useful for those who want to consume Tanium APIs, but is also useful for anyone wanting to pull from an API that may not instantly return a full set of results, and then when the data is ready, to pull the data using pagination.

 
# Flow Beginning

The flow shows two data subject areas. The data subjects are determined by the Tanium question and it is assumed that saved questions have been defined within the Tanium source. Since the data results are combined into a funnel and all processed in the same way (due to the fact that the Tanium API always returns results with the same structure), additional data subjects could be added in the same way and joined into the funnel.

Each data subject is pulling from three different Tanium endpoints (DEV, TEST, PROD environments). This is to simulate having three domains and needing to pull information from all three. There is also a processor for sending a test flow file that can be used to test the flow without invoking an API call. All of these initial processors are of type GenerateFlowFile and are used to set which Tanium environment is to be queried, what saved question id is to be used (or in the case Test FlowFile processors, the fact that it is just a test), and this information is stored within flowfile attributes to be used further along in the flow. Having separated initial GenerateFlowFile processors in this manner allows for them to be separately and independently triggered on any desired schedule. This portion of the flow is shown below in the Cloudera DataFlow Designer.
![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/7bc68f25-925c-46d1-9333-2f018d4efa41)

As can be seen above, before the flow goes into the funnel, there are processors that allow adding attributes that are specific to the data subject; in this case they are defining the target table name.

 
# API or Test

Next a RouteOnAttribute determines whether the process was triggered by a Test FlowFile processor or one of the processors representing Tanium DEV, TEST, or PROD. If it is from a Test FlowFile processor, the API call is bypassed.
![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/51b55f63-7660-46d3-a18c-65d7f7fb5a19)

 
# Tanium API calls

The Tanium API Call process group starts off with setting the base URL of the API endpoint based upon the Tanium environment.
![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/63e7ada0-a7b8-4189-910d-5cdf0fd5e04e)

 

It also initializes a results loop count which will be discussed later on. But first comes the login portion.

 
# Login To API

The API login is in its own processor group. This is done to allow for security of limiting access to this group to only those that need to know the credentials. Why not use sensitive properties for this? Well the Tanium API requires a POST request with the username and password contained in JSON in the body. However, the InvokeHTTP processor supplies the body request from the flowfile contents, which of course cannot be made sensitive. Currently I have a feature request open for the need to be able to set the body content with sensitive values and the moderators agreed in the
comments that it is something that should be incorporated in a future release of the InvokeHTTP processor. (More details at https://issues.apache.org/jira/projects/NIFI/issues/NIFI-11331?filter=allopenissues)
![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/be3e62ab-0d78-43c4-b03b-a9a83ddf7248)

There is also logic to retry on alternate backup Tanium endpoints if the main endpoint fails.

 
# Looping Until Query Has Completed

Tanium questions can take some time to complete, depending upon query complexity and number of devices in the environment. In order to determine whether a query is complete, an API call is made to start the question query running, but just return the info about the questions query status. Then there are two data points that are compared: "tested" and "estimated-total" and when those closely match, it moves on to the next set of processors that control paging of results.
![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/07908ff0-f35f-40cb-9335-ad1d5e5e0c2d)

 
# Paging Results

After the results are received, a call to the API to get the actual results is made. Passed into this request are the start and count values in order to page the results back.Row, estimated, and tested totals are compared as well as looking at the row array to see if data is there. All of these factors determine whether data is being returned, and, if so, that data is captured and another request is made. Even when those factors indicate that the data pull has completed, a loop counter with delay is used to ensure that there are not a few more rows that were coming in on the tail end of the question results.After that, any results with empty rows are discarded and each resulting flowfile gets a API UUID assigned to it that is used to correlate merging after it gets split during the procedure of extracting column information.
![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/1311212e-12c5-4492-a347-b4b9906b3e8d)

 
# Tanium Result Structure

Before we go further, you may be wondering about the statement above about extracting column information. What does a Tanium data result look like? Well, it has all of the information that the informational pull had as well as the actual results, which includes a section for column names and a section for the results those correspond to. However, the keys for the values in the results are all the word "text". So somehow the column names need to be matched up with the rows and brought together into a form that can be used in recordset aware processors. Below is an example result (abbreviated to show only relevant data) with just one row:

![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/b3307062-cb5d-4955-ab2b-88e8a99e2b66)

 
# Get Column Names

The result is sent through a processor group that uses an EvaluateJsonPath to extract the Column portion of the results. Then various JoltTransformJSON and ReplaceText processors are used to turn that into comma separated results to allow for a succinct list of the column names to be used later on. Once we have the column names stored in a flowfile attribute, we use a ModifyBytes processor to remove the content. Since we still need to get the row data, the original flow files are sent along a parallel path and Wait/Notify processors are used to ensure they move on to the next step at the same time. As they leave the Get Columns processor group, they are merged back together, using the API GUID mentioned earlier as the correlation. This gets the new column list attribute combined with the original flowfile content.
![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/0720142a-cc2b-4f27-a47d-4a1bdbf900f7)

 
# Getting Row Values

As stated earlier row data consists of values that all have the key "text" in them, with the only relation to the column names being that they are in the same order. Each row has an id key in it. So two loops are used. Before they are started, the column list is copied into another variable that is used to keep track which column is being worked on. Then the outer loop uses Regex to find the first "id" key value. An inner loop then uses Regex and also ExecuteGroovyScript processors to traverse each instance of the "text" key and replace that with the corresponding column name from the column variable and separating it from the value with a colon. That column variable then has that column removed from it and the next column name becomes the current column until all column names have been removed. Once the "text" keys have all been replaced, then the "id" key is replaced with the word "replaced". When there are no "id" keys left, then it moves on to cleaning up all the data until only the column names and row values are in the data in a JSON format that can be loaded by recordset aware processors.
![image](https://github.com/vincelombardo/NiFiTaniumToPutDatabaseRecord/assets/21046032/b5655731-a95c-4465-bfe8-8e1abf3c9133)

 
# Merging and Loading

Once the data leaves the Get Row Values processor group, it is ready to be loaded into a destination. First it feeds into a MergeContent processor so that multiple entries can be combined into a single, larger batch. Then any recordset aware processor can use a JSONTreeReader record reader and load the data into its destination.

See the Tanium_to_PutDatabaseRecord template files in this repository for full details.
