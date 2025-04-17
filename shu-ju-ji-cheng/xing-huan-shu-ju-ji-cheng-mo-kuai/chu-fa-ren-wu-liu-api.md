# 触发任务流API

```python
import time
import requests
import logging

# 忽略 HTTPS 警告
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# 配置 logger
logging.basicConfig(
    level=logging.INFO,  # 设置日志级别为 INFO
    format="%(asctime)s - %(levelname)s - %(message)s",  # 设置日志格式
)

# 获取 Token
def get_token():
    url = "https://10.149.177.208:28190/oauth/token"
    params = {
        "client_id": "app",
        "client_secret": "secret",
        "username": "admin",
        "password": "admin",
        "grant_type": "password",
    }
    try:
        response = requests.post(url, params=params, verify=False)
        response.raise_for_status()
        logging.info("Token obtained successfully.")
        return response.json()["access_token"]
    except requests.RequestException as e:
        logging.error(f"Failed to get token: {e}")
        raise

# 启动任务
def start_task(token):
    url = "http://10.149.177.211:28911/studio/api/workflow/v1/flows/949f294572a94d538edad7cf80e83686/actions/manual"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json;charset=UTF-8"
    }
    payload = {
        "globalParameters": [],
        "workspaceParameters": [],
        "flowParameters": [],
        "taskParameters": [],
        "operationScope": "SELECTED"
    }
    try:
        response = requests.post(url, headers=headers, json=payload, verify=False)
        response.raise_for_status()
        logging.info("Task started successfully.")
        return response.json()
    except requests.RequestException as e:
        logging.error(f"Failed to start task: {e}")
        raise

# 查询任务状态
def check_task_status(token, run_id):
    url = f"http://10.149.177.211:28911/studio/api/workflow/v1/flowExecutions/{run_id}"
    headers = {
        "Authorization": f"Bearer {token}"
    }
    while True:
        try:
            response = requests.get(url, headers=headers, verify=False)
            response.raise_for_status()
            result = response.json()
            execution_state = result.get("executionState")
            logging.info(f"Current Execution State: {execution_state}")

            if execution_state != "FES_RUNNING" and execution_state != "FES_WAITING":
                logging.info("Task completed.")
                return result
        except requests.RequestException as e:
            logging.error(f"Error checking task status: {e}")
            raise

        time.sleep(10)

# 主程序
if __name__ == "__main__":
    try:
        logging.info("Getting token...")
        token = get_token()

        logging.info("Starting task...")
        run_id = start_task(token)

        logging.info("Checking task status...")
        final_result = check_task_status(token, run_id)
        logging.info("Task completed. Final result:")
        logging.info(final_result)

    except Exception as e:
        logging.error(f"An error occurred: {e}")
```
