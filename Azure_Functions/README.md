# Azure Function sample

## Example
This example uses a sample in a blob storage account and saves the summary in a storage account.

- It uses the Azure Foundry SDK
- It uses Managed Identity for the Function (Cognitive Services User)

```python
import requests
from azure.storage.blob import BlobServiceClient
from azure.identity import DefaultAzureCredential
import azure.functions as func

# Azure Blob Storage details
STORAGE_ACCOUNT_URL = "https://yourstorageaccount.blob.core.windows.net"
CONTAINER_NAME = "your-container"
BLOB_NAME = "your-document.txt"

# Azure AI Foundry Model Endpoint
AI_FOUNDY_ENDPOINT = "https://your-foundry-instance.azure.com/v1/deployments/gpt4o/completions"

def main(req: func.HttpRequest) -> func.HttpResponse:
    try:
        # Authenticate using Managed Identity
        credential = DefaultAzureCredential()
        blob_service_client = BlobServiceClient(account_url=STORAGE_ACCOUNT_URL, credential=credential)
        blob_client = blob_service_client.get_blob_client(CONTAINER_NAME, BLOB_NAME)

        # Read the document content
        document_content = blob_client.download_blob().readall().decode("utf-8")

        # Send to GPT-4o for summarization
        headers = {
            "Authorization": f"Bearer {credential.get_token(AI_FOUNDY_ENDPOINT + '/.default').token}",
            "Content-Type": "application/json"
        }
        data = {
            "model": "gpt-4o",
            "messages": [{"role": "user", "content": f"Summarise this document into bullet points:\n\n{document_content}"}],
            "temperature": 0.5
        }

        response = requests.post(AI_FOUNDY_ENDPOINT, headers=headers, json=data)

        if response.status_code == 200:
            summary = response.json()["choices"][0]["message"]["content"]
            return func.HttpResponse(summary, mimetype="text/plain")
        else:
            return func.HttpResponse(f"Error: {response.text}", status_code=response.status_code)

    except Exception as e:
        return func.HttpResponse(f"Error: {str(e)}", status_code=500)
```

You can also incorporate prompt flow

```python
import requests
from azure.storage.blob import BlobServiceClient
from azure.identity import DefaultAzureCredential
import azure.functions as func

# Azure Blob Storage details
STORAGE_ACCOUNT_URL = "https://yourstorageaccount.blob.core.windows.net"
CONTAINER_NAME = "your-container"
INPUT_BLOB_NAME = "your-document.txt"
OUTPUT_BLOB_NAME = "summarized-document.txt"

# Azure Prompt Flow API details
PROMPT_FLOW_URL = "https://your-aml-instance.azure.com/promptflow/deployments/your-flow/run"

def main(req: func.HttpRequest) -> func.HttpResponse:
    try:
        # Authenticate using Managed Identity
        credential = DefaultAzureCredential()
        blob_service_client = BlobServiceClient(account_url=STORAGE_ACCOUNT_URL, credential=credential)

        # Read the document from Blob Storage
        blob_client = blob_service_client.get_blob_client(CONTAINER_NAME, INPUT_BLOB_NAME)
        document_content = blob_client.download_blob().readall().decode("utf-8")

        # Reference variant2 instead of hardcoding the prompt
        prompt_variant = "variant2"  # This tells Prompt Flow to use the predefined long prompt

        # Send content to Prompt Flow for processing
        token = credential.get_token("https://your-aml-instance.azure.com/.default").token
        headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
        payload = {"inputs": {"user_prompt": prompt_variant, "document": document_content}}

        response = requests.post(PROMPT_FLOW_URL, headers=headers, json=payload)

        if response.status_code == 200:
            summary = response.json()["outputs"]["summary"]

            # Save the summary back to Blob Storage
            output_blob_client = blob_service_client.get_blob_client(CONTAINER_NAME, OUTPUT_BLOB_NAME)
            output_blob_client.upload_blob(summary, overwrite=True)

            return func.HttpResponse(f"Summary saved successfully!", mimetype="text/plain")

        else:
            return func.HttpResponse(f"Error: {response.text}", status_code=response.status_code)

    except Exception as e:
        return func.HttpResponse(f"Error: {str(e)}", status_code=500)

```