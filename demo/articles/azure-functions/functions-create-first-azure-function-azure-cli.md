
# <a name="quickstart-create-your-first-function-from-the-command-line-using-azure-cli"></a>快速入門：使用 Azure CLI 從命令列建立您的第一個函式

此快速入門主題會逐步引導您從命令列或終端機建立您的第一個函式。 您會使用 Azure CLI 來建立函數應用程式，此應用程式是主控函式的[無伺服器](https://azure.microsoft.com/solutions/serverless/)基礎結構。 函式程式碼專案是使用 [Azure Functions Core Tools](functions-run-local.md) (也用來將函式應用程式專案部署到 Azure) 從範本產生的。

您可以使用 Mac、Windows 或 Linux 電腦，依照下面步驟操作。

## <a name="prerequisites"></a>Prerequisites

在執行此範例之前，您必須具備下列項目︰

+ 安裝 [Azure Functions Core Tools](./functions-run-local.md#v2) 2.6.666 版或更新版本。

+ 安裝 [Azure CLI](/cli/azure/install-azure-cli)。 此文章需要 Azure CLI 2.0 版或更新版本。 執行 `az --version` 以尋找您擁有的版本。 您也可以使用 [Azure Cloud Shell](https://shell.azure.com/bash)。

+ 有效的 Azure 訂用帳戶。

## <a name="create-the-local-function-app-project"></a>建立本機函式應用程式專案

從命令列執行下列命令，以在目前本機目錄的 `MyFunctionProj` 資料夾中建立函式應用程式專案。 系統也會在 `MyFunctionProj` 中建立 GitHub 存放庫。

```command
func init MyFunctionProj
```

當出現提示時，請從下列語言選項中選取背景工作角色執行階段：

+ `dotnet`：建立 .NET 類別庫專案 (.csproj)。

專案建立後，請使用下列命令瀏覽至新的 `MyFunctionProj` 專案資料夾。

```command
cd MyFunctionProj
```


## <a name="enable-extension-bundles"></a>啟用延伸模組搭售方案

安裝繫結擴充功能最簡單的方式是啟用擴充功能套件組合。 當您啟用套件組合時，系統會自動安裝一組預先定義的擴充功能套件。

若要啟用擴充功能套件組合，請開啟 host.json 檔案，並更新其內容以符合下列程式碼：

```
{
    "version": "2.0",
    "extensionBundle": {
        "id": "Microsoft.Azure.Functions.ExtensionBundle",
        "version": "[1.*, 2.0.0)"
    }
}
```
## <a name="create-a-function"></a>建立函式

下列命令會建立名為 `MyHttpTrigger` 的 HTTP 觸發函式。

```command
func new --name MyHttpTrigger --template "HttpTrigger"
```

當命令執行時，您會看到如下輸出：

```output
The function "MyHttpTrigger" was created successfully from the "HttpTrigger" template.
```

## <a name="run-the-function-locally"></a>在本機執行函式

下列命令會啟動函數應用程式。 應用程式使用的 Azure Functions 執行階段和 Azure 中的是相同的。 視您的專案語言而定，啟動命令會有所不同。

### <a name="c"></a>C\#

```command
func start --build
```

當 Functions 主機啟動時，它會寫入類似下列輸出的內容 (已截斷來提高可讀性)：

```output

                  %%%%%%
                 %%%%%%
            @   %%%%%%    @
          @@   %%%%%%      @@
       @@@    %%%%%%%%%%%    @@@
     @@      %%%%%%%%%%        @@
       @@         %%%%       @@
         @@      %%%       @@
           @@    %%      @@
                %%
                %

...

Content root path: C:\functions\MyFunctionProj
Now listening on: http://0.0.0.0:7071
Application started. Press Ctrl+C to shut down.

...

Http Functions:

        HttpTrigger: http://localhost:7071/api/MyHttpTrigger

[8/27/2018 10:38:27 PM] Host started (29486ms)
[8/27/2018 10:38:27 PM] Job host started
```

從執行階段輸出複製 `HttpTrigger` 函式的 URL，並將它貼到瀏覽器的網址列。 將查詢字串 `?name=<yourname>` 附加至此 URL 並執行要求。 下圖顯示瀏覽器中對於本機函式傳回之 GET 要求所做出的回應︰


您現在已經在本機執行您的函式，您可以開始在 Azure 中建立函數應用程式與其他必要資源。


下列範例會建立名為 `myResourceGroup` 的資源群組。  
如果您未使用 Cloud Shell，請先使用 `az login` 登入。

```azurecli-interactive
az group create --name myResourceGroup --location eastus
```
## <a name="create-an-azure-storage-account"></a>建立 Azure 儲存體帳戶

函式會使用 Azure 儲存體中的一般用途帳戶來維護函式的狀態和其他資訊。 在使用 [az storage account create](/cli/azure/storage/account#az-storage-account-create) 命令所建立的資源群組中建立一般用途的儲存體帳戶。

在下列命令中，使用全域唯一儲存體帳戶名稱來替代您看見 `<storage_name>` 預留位置的地方。 儲存體帳戶名稱必須介於 3 到 24 個字元的長度，而且只能包含數字和小寫字母。

```azurecli-interactive
az storage account create --name <storage_name> --location eastus --resource-group myResourceGroup --sku Standard_LRS
```


## <a name="create-a-function-app"></a>建立函數應用程式

您必須擁有函式應用程式以便主控函式的執行。 函式應用程式會提供環境來讓您的函式程式碼進行無伺服器執行。 其可讓您將多個函式群組為邏輯單位，以方便您管理、部署、調整和共用資源。 使用 [az functionapp create](/cli/azure/functionapp#az-functionapp-create) 命令來建立函式應用程式。

在下列命令中，使用唯一函式應用程式名稱來替代您看見 `<APP_NAME>` 預留位置的地方，並使用儲存體帳戶名稱來替代 `<STORAGE_NAME>`。 `<APP_NAME>` 會作為函式應用程式的預設 DNS 網域，所以此名稱在 Azure 的所有應用程式中都必須是唯一的名稱。 您也應該從 `dotnet` (C#) 或 `node` (JavaScript)，設定函式應用程式的 `<language>` 執行階段。

```azurecli-interactive
az functionapp create --resource-group myResourceGroup --consumption-plan-location eastus \
--name <APP_NAME> --storage-account  <STORAGE_NAME> --runtime dotnet
```

設定 _consumption-plan-location_ 參數，即表示函數應用程式是裝載於取用主控方案。 在這個無伺服器方案中，會根據您的函式，視需要動態新增資源，且只有在函式執行時才需要付費。

建立函式應用程式後，Azure CLI 會顯示類似下列範例的資訊：

```json
{
  "availabilityState": "Normal",
  "clientAffinityEnabled": true,
  "clientCertEnabled": false,
  "containerSize": 1536,
  "dailyMemoryTimeQuota": 0,
  "defaultHostName": "quickstart.azurewebsites.net",
  "enabled": true,
  "enabledHostNames": [
    "quickstart.azurewebsites.net",
    "quickstart.scm.azurewebsites.net"
  ],
   ....
    // Remaining output has been truncated for readability.
}
```
## <a name="deploy-the-function-app-project-to-azure"></a>將函式應用程式專案部署至 Azure

在 Azure 中建立函式應用程式之後，您可以使用 Core 工具命令來將專案程式碼部署至 Azure。 在這些範例中，請將 `<APP_NAME>` 取代為上一個步驟中您的應用程式名稱。

### <a name="c--javascript"></a>C\# / JavaScript

```command
func azure functionapp publish <APP_NAME>
```


您會看到類似下列的輸出，該輸出已截短，以提高可讀性：

```output
Getting site publishing info...
...

Preparing archive...
Uploading content...
Upload completed successfully.
Deployment completed successfully.
Syncing triggers...
Functions in myfunctionapp:
    HttpTrigger - [httpTrigger]
        Invoke url: https://myfunctionapp.azurewebsites.net/api/httptrigger?code=cCr8sAxfBiow548FBDLS1....
```

複製 `HttpTrigger` 的叫用 `Invoke url` 值，您現在可以使用它來測試 Azure 中的功能。 該 URL 包含一個 `code` 查詢字串值，它是您的函式金鑰。 此金鑰使其他人很難在 Azure 中呼叫 HTTP 觸發程序端點。

## <a name="test"></a>驗證 Azure 中的函式

您可以使用網頁瀏覽器來驗證已部署的函式。  請將 URL (包括函式索引鍵) 複製到網頁瀏覽器的位址列中。 請先將查詢字串 `&name=<yourname>` 附加至 URL，然後再執行要求。

![使用網頁瀏覽器來呼叫函式。](https://docs.microsoft.com/zh-tw/azure/includes/media/functions-test-function-code/functions-azure-cli-function-test-browser.png)  

或者，您可以使用 cURL 來驗證已部署的函式。 使用您從上一個步驟複製的 URL (包括函式索引鍵)，將查詢字串 `&name=<yourname>` 附加至 URL。

![使用 cURL 來呼叫 Azure 中的函式。](https://docs.microsoft.com/zh-tw/azure/includes/media/functions-test-function-code/functions-azure-cli-function-test-curl.png) 

## <a name="clean-up-resources"></a>清除資源

此集合中的其他快速入門會以本快速入門為基礎。 如果您打算繼續進行後續的快速入門或教學課程，請勿清除在此快速入門中建立的資源。 如果您不打算繼續，請使用下列命令來刪除本快速入門中建立的所有資源：

```azurecli-interactive
az group delete --name myResourceGroup
```
出現提示時選取 `y`。