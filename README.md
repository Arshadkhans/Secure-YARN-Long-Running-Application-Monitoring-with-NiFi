Secure YARN Long Running Application Monitoring with NiFi


Short Description:
Simple NiFi flow to monitor and Alert on a Long Running YARN Application.

Introduction

•	Here is a small demo how NiFi can help you monitor and alert on Long Running YARN Applications.
•	You can download the flow XML I have created here

Prerequisite
•	Make sure you have your CDF Cluster up and running. 
Steps:
1.	Assuming you have NiFi UI up and Available, lets drop invokeHTTP processor to pull data from YARN REST API:
Configure Processor with URL given as below, which pulls all Applications in Running state. resource-manager-server.com is my Resource Manager running on 8090 port:
https://resource-manager-server.com:8090/ws/v1/cluster/apps?states=running
 
![image](https://user-images.githubusercontent.com/12103876/215949357-20b72409-adff-4b48-b651-b40c2abf358f.png)
![image](https://user-images.githubusercontent.com/12103876/215949371-f226e6ce-85dc-4d66-8e41-a39a97f4bd38.png)


StandardRestrictedSSLContextService for SSL configuration go to controller service and specify the path of it. 
/opt/cloudera/security/keystore/truststore.jks
And password 
![image](https://user-images.githubusercontent.com/12103876/215949502-cc0aab28-10ea-4b9c-871b-16d42b036731.png)
  KeytabCredentilalsservice:
  ![image](https://user-images.githubusercontent.com/12103876/215949545-9a379bf2-097a-4c4a-9698-93f6b71c7cea.png)
Lets schedule the processor to run only every 10sec so that you don’t query too often.
2.	To make sure the json file received is sent only down stream if its not empty (i.e no jobs are running on the cluster), add a RouteText processor to check null in the content as below:

![image](https://user-images.githubusercontent.com/12103876/215949635-3c9a6c47-1682-4896-a223-e559f3dd7a3a.png)
Route on unmatched relation, only when json is not empty. Auto terminate null connection created above. Connect InvokeHTTP processor to RouteText for success relation
3.	As the Rest call outputs the application details in Json format, lets use a SplitJson processor to separate individualapplication details.
Provide “JsonPath Expression” value as  “$.apps.app”  in the configuration.
![image](https://user-images.githubusercontent.com/12103876/215949697-3071ad3e-5a26-4adb-bd89-88dcec3550a4.png)

4.	Connect RouteText to SplitJson for 'unmatched' relation and auto terminate 'null' relation.
5.	Lets add EvaluateJsonPath processor to extract required fields and add them to flow-file attribute: Configure it as below:

![image](https://user-images.githubusercontent.com/12103876/215949801-99342436-5f24-4d30-a9dc-07ed61b765dc.png)

Extracted Attribute 'ElapsedTime' is the key player here which tell how long the application was running.
6.	Connect SplitJson to EvaluateJsonPath for 'split' relation.
7.	Add a RouteOnAttribute processor to the canvas to check value of 'ElapsedTime', here lets check and alert on all application passed 1 Hour and 10 hours.
10hr 	: ${ElapsedTime:gt(36000000)}
1hr 	: ${ElapsedTime:gt(3600000)}

![image](https://user-images.githubusercontent.com/12103876/215949865-424e59dc-9290-478e-84d7-83070735e981.png)

8.	Create and start two sets of DistributedMapCache controller services: 
9.	2 DistributedMapCacheClientService, 2 DistributedMapCacheServer for 1hr jobs and 10hr jobs so that we keep track of all the applications crossed 1hr and 10hr and no duplicate alerts for same are sent.

![image](https://user-images.githubusercontent.com/12103876/215949942-29486911-42b8-427c-bdc8-5304e9608fb3.png)

Make sure those DistributedMapCacheServer run on different ports
7.	Add a two PutDistributedMapCache processors to update the cache with jobs ran past 1hr and 10hrs respectively. Configure it as below adding Distributed cache service.

![image](https://user-images.githubusercontent.com/12103876/215950000-e2909f2a-0d07-43cb-9255-8c446e7b6f96.png)
8.	Lets auto terminate Failure relationship and connect success relationship to PutEmail processor which will sent out email for any new Yarn Application which crossed the threshold of 1hr and 10hr .
9.	Make sure you have formatted the email body and subject to have all information about the failed job:
![image](https://user-images.githubusercontent.com/12103876/215950077-46d2827d-b03e-4581-aa03-094d72bc3129.png)
10.	Auto terminate success and failure relationship for PutEmail processor. Once flow is completed, it would look something like below:
![image](https://user-images.githubusercontent.com/12103876/215950191-c149afb6-a3a5-434d-9b6d-667b42d2caf6.png)

 
11.	Once you start the Flow, you will get alerts for each Yarn application which cross the threshold set which is 1hr and 10hr. My Email Alert would look like below:

![image](https://user-images.githubusercontent.com/12103876/215950236-8ad1618d-e1c7-4560-a37f-b3d78ad84493.png)

