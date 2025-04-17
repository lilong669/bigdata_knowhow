# 触发批量任务API

DS支持导出WSDL, 然后你需要做的就是导入到soap ui 进行解析，然后测试，之后就可以编写调用代码了。

```python
import requests
import re
import time
import logging

# Set up the logger
logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)  # Set the log level to DEBUG
ch = logging.StreamHandler()  # Output logs to console
ch.setLevel(logging.DEBUG)  # Set the handler level to DEBUG
formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')
ch.setFormatter(formatter)
logger.addHandler(ch)

# URL of the SOAP service
url = "http://10.149.2.145:8080/DataServices/servlet/webservices?ver=2.1"

# SOAPAction header
run_batch_headers = {
    "SOAPAction": "jobAdmin=Run_Batch_Job",
    "Content-Type": "application/xml"
}
# SOAP request body (without XML declaration)
run_batch_request = """
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ser="http://www.businessobjects.com/DataServices/ServerX.xsd">
   <soapenv:Header/>
   <soapenv:Body>
      <ser:RunBatchJobRequest>
         <jobName>CWGK_ZZJG_JOB1</jobName>
         <repoName>Repo_cgdgdsp</repoName>
      </ser:RunBatchJobRequest>
   </soapenv:Body>
</soapenv:Envelope>
"""
# SOAPAction header
get_batch_status_headers = {
    "SOAPAction": "jobAdmin=Get_BatchJob_Status",
    "Content-Type": "application/xml"
}
# SOAP request body (without XML declaration)
get_batch_status_request = """
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ser="http://www.businessobjects.com/DataServices/ServerX.xsd">
   <soapenv:Header/>
   <soapenv:Body>
      <ser:batchJobStatusRequest>
         <runID>{run_batch_id}</runID>
         <repoName>Repo_cgdgdsp</repoName>
      </ser:batchJobStatusRequest>
   </soapenv:Body>
</soapenv:Envelope>
"""


# Function to get batch job status
def get_batch_job_status(rid_value):
    get_batch_status_request_filled = get_batch_status_request.format(run_batch_id=rid_value)
    logger.debug(f"Sending batch job status request with runID: {rid_value}")
    get_batch_status_response = requests.post(url, data=get_batch_status_request_filled,
                                              headers=get_batch_status_headers)

    if get_batch_status_response.status_code == 200:
        match = re.search(r'<status>(\w+)</status>', get_batch_status_response.text)
        if match:
            return match.group(1)
    else:
        logger.error(f"Request failed with status code: {get_batch_status_response.status_code}")
        logger.error(f"Response content: {get_batch_status_response.text}")
    return None


# Start the batch job
logger.debug("Sending run batch job request")
run_batch_response = requests.post(url, data=run_batch_request, headers=run_batch_headers)

# Check if the request was successful
if run_batch_response.status_code == 200:
    # Extract the run id (rid)
    match = re.search(r'<rid>(\d+)</rid>', run_batch_response.text)
    if match:
        rid_value = match.group(1)
        logger.info(f"Extracted rid value: {rid_value}")

        # Check the job status in a loop every 10 seconds
        while True:
            logger.debug(f"Checking status for runID: {rid_value}")
            status = get_batch_job_status(rid_value)
            if status is None:
                logger.error("Error getting job status")
                break

            logger.info(f"Current status: {status}")
            if status in ["succeeded", "error"]:
                logger.info(f"Final status: {status}")
                break

            # Wait for 10 seconds before checking again
            time.sleep(10)
    else:
        logger.error("No rid value found in the response")
else:
    logger.error(f"Request failed with status code: {run_batch_response.status_code}")
    logger.error(f"Response content: {run_batch_response.text}")  # Log the full response for debugging
```

```python
# 简易版本，只是触发
import requests
import re

# SOAP request body (without XML declaration)
soap_request = """
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ser="http://www.businessobjects.com/DataServices/ServerX.xsd">
   <soapenv:Header/>
   <soapenv:Body>
      <ser:RunBatchJobRequest>
         <jobName>CWGK_ZZJG_JOB1</jobName>
         <repoName>Repo_cgdgdsp</repoName>
      </ser:RunBatchJobRequest>
   </soapenv:Body>
</soapenv:Envelope>
"""

# URL of the SOAP service
url = "http://10.149.2.145:8080/DataServices/servlet/webservices?ver=2.1"

# SOAPAction header
headers = {
    "SOAPAction": "jobAdmin=Run_Batch_Job",
    "Content-Type": "application/xml"
}

# Send the POST request
response = requests.post(url, data=soap_request, headers=headers)

# Check if the request was successful
if response.status_code == 200:
    # Use regular expression to extract the value between <rid> and </rid>
    match = re.search(r'<rid>(\d+)</rid>', response.text)
    if match:
        rid_value = match.group(1)
        print(f"Extracted rid value: {rid_value}")
    else:
        print("No rid value found")
else:
    print(f"Request failed with status code: {response.status_code}")
    print(f"Response content: {response.text}")  # Log the full response for debugging
```
