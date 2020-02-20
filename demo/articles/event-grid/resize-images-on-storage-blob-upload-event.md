
# <a name="tutorial-automate-resizing-uploaded-images-using-event-grid"></a>教學課程：使用 Event Grid 自動調整已上傳映像的大小

Azure Event Grid 是一項雲端事件服務。 Event Grid 可讓您建立由 Azure 服務或協力廠商資源引發之事件的訂閱。  

本教學課程是一系列儲存體教學課程的第二部分。 它延伸了[上一個儲存體教學課程][previous-tutorial]，以使用 Azure Event Grid 與 Azure Functions 新增無伺服器自動縮圖產生。 Event Grid 可讓 Azure Function 回應 Azure Blob 儲存體 事件，並產生上傳映像的縮圖。 針對 Blob 儲存體建立事件建立事件訂閱。 當 blob 加入特定的 Blob 儲存體容器時，會呼叫函式端點。 從 Event Grid 傳遞至函式繫結的資料用於存取 Blob 並產生縮圖映像。

您可以使用 Azure CLI 與 Azure 入口網站，將調整大小功能加入現有的映像上傳應用程式。

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)

![瀏覽器中已發佈的 Web 應用程式](https://docs.microsoft.com/zh-tw/azure/event-grid/media/resize-images-on-storage-blob-upload-event/tutorial-completed.png)

---

在本教學課程中，您會了解如何：

> * 建立一般的 Azure 儲存體帳戶
> * 使用 Azure Functions 部署無伺服器程式碼
> * 在 Event Grid 中建立 Blob 儲存體事件訂閱

## <a name="prerequisites"></a>必要條件

若要完成本教學課程：


如果您先前未在訂用帳戶中註冊事件方格資源提供者，請務必註冊。

```azurecli-interactive
az provider register --namespace Microsoft.EventGrid
```

如果您選擇在本機安裝和使用 CLI，本教學課程需要 Azure CLI 2.0.14 版或更新版本。 執行 `az --version` 以尋找版本。 如果您需要安裝或升級，請參閱[安裝 Azure CLI]( /cli/azure/install-azure-cli)。 

如果您未使用 Cloud Shell，您必須先使用 `az login` 登入。

## <a name="create-an-azure-storage-account"></a>建立 Azure 儲存體帳戶

Azure Functions 需要一般的儲存體帳戶。 除了您在上一個教學課程所建立的 Blob 儲存體帳戶外，請再使用 [az storage account create](/cli/azure/storage/account) 命令，於資源群組中另外建立一般的儲存體帳戶。 儲存體帳戶名稱必須介於 3 到 24 個字元的長度，而且只能包含數字和小寫字母。 

1. 設定一個變數，用以保存您在上一個教學課程中建立的資源群組名稱。 

    ```azurecli-interactive
    resourceGroupName=myResourceGroup
    ```
2. 針對 Azure Functions 所需新儲存體帳戶的名稱來設定變數。 

    ```azurecli-interactive
    functionstorage=<name of the storage account to be used by the function>
    ```
3. 建立 Azure 函式的儲存體帳戶。 

    ```azurecli-interactive
    az storage account create --name $functionstorage --location eastus \
    --resource-group $resourceGroupName --sku Standard_LRS --kind storage
    ```

## <a name="create-a-function-app"></a>建立函數應用程式  

您必須擁有函式應用程式以便主控函式的執行。 函式應用程式會提供環境來讓您的函式程式碼進行無伺服器執行。 使用 [az functionapp create](/cli/azure/functionapp) 命令來建立函式應用程式。 

在下列命令中，請提供您自己的唯一函式應用程式名稱。 函式應用程式會作為函式應用程式的預設 DNS 網域，所以此名稱在 Azure 的所有應用程式中都必須是唯一的名稱。 

1. 指定要建立的函式應用程式名稱。 

    ```azurecli-interactive
    functionapp=<name of the function app>
    ```
2. 建立 Azure 函式。 

    ```azurecli-interactive
    az functionapp create --name $functionapp --storage-account $functionstorage \
    --resource-group $resourceGroupName --consumption-plan-location eastus
    ```

現在，您必須設定函式應用程式，才能連線到您在[上一個教學課程][previous-tutorial]中建立的 Blob 儲存體帳戶。

## <a name="configure-the-function-app"></a>設定函式應用程式

此函式需要 Blob 儲存體帳戶的認證，認證會使用 [az functionapp config appsettings set](/cli/azure/functionapp/config/appsettings) 命令來新增至函式應用程式的應用程式設定。

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)

```azurecli-interactive
可以先 
echo $blobStorageAccount 確認一下之前設置的還在

blobStorageAccount=<name of the Blob storage account you created in the previous tutorial>

storageConnectionString=$(az storage account show-connection-string --resource-group $resourceGroupName \
--name $blobStorageAccount --query connectionString --output tsv)

az functionapp config appsettings set --name $functionapp --resource-group $resourceGroupName \
--settings AzureWebJobsStorage=$storageConnectionString THUMBNAIL_CONTAINER_NAME=thumbnails \
THUMBNAIL_WIDTH=100 FUNCTIONS_EXTENSION_VERSION=~2
```

---

`FUNCTIONS_EXTENSION_VERSION=~2` 設定會讓函式應用程式在 2.x 版的 Azure Functions 執行階段上執行。

您現在可以將函式程式碼專案部署到此函式應用程式。

## <a name="deploy-the-function-code"></a>部署函式程式碼 

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)

範例 C# 大小調整函式可從 [GitHub](https://github.com/Azure-Samples/function-image-upload-resize) 取得。 使用 [az functionapp deployment source config](/cli/azure/functionapp/deployment/source) 命令，將此程式碼專案部署至函式應用程式。 

```azurecli-interactive
az functionapp deployment source config --name $functionapp --resource-group $resourceGroupName --branch master --manual-integration --repo-url https://github.com/Azure-Samples/function-image-upload-resize
```

---

影像大小調整函式是由從 Event Grid 服務傳送給它的 HTTP 要求所觸發。 您可藉由建立事件訂閱，告訴 Event Grid 您想要在您函式的 URL 取得這些通知。 在此教學課程中，您會訂閱 Blob 建立的事件。

從 Event Grid 通知傳遞給函式的資料包括 Blob 的 URL。 該 URL 接著會傳遞至輸入繫結，以從 Blob 儲存體獲取上傳的映像。 此函式會產生縮圖映像，並將產生的串流寫入 Blob 儲存體中的個別容器。 

此專案使用 `EventGridTrigger` 作為觸發程序類型。 建議透過一般 HTTP 觸發程序使用 Event Grid 觸發程序。 Event Grid 會自動驗證 Event Grid 函式的觸發程序。 若要使用 HTTP 觸發程序，您必須實作驗證回應

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)

若要深入了解此函式，請參閱 [function.json 和 run.csx 檔案](https://github.com/Azure-Samples/function-image-upload-resize/tree/master/ImageFunctions)。

---

函式專案程式碼會直接從公用範例存放庫部署。 若要深入了解 Azure Functions 的部署選項，請參閱[Azure Functions 的持續部署](../azure-functions/functions-continuous-deployment.md)。

## <a name="create-an-event-subscription"></a>建立事件訂閱

事件訂閱表示您想要傳送至特定端點之提供者產生的事件。 在此情況下，端點會由函式公開。 使用下列步驟，在 Azure 入口網站中建立事件訂閱，以傳送通知給您的函式： 

1. 在 [Azure 入口網站](https://portal.azure.com)中，選取左側功能表中的 [所有服務]  ，然後選取 [函式應用程式]  。 

    ![瀏覽至 Azure 入口網站中的函式應用程式](https://docs.microsoft.com/zh-tw/azure/event-grid/media/resize-images-on-storage-blob-upload-event/portal-find-functions.png)

2. 展開函式應用程式，選擇 **Thumbnail** 函式，然後選取 [新增事件方格訂用帳戶]  。

    ![瀏覽至 Azure 入口網站中的函式應用程式](https://docs.microsoft.com/zh-tw/azure/event-grid/media/resize-images-on-storage-blob-upload-event/add-event-subscription.png)

3. 使用表格中指定的事件訂閱設定。
    
    ![從 Azure 入口網站中的函式建立事件訂閱](https://docs.microsoft.com/zh-tw/azure/event-grid/media/resize-images-on-storage-blob-upload-event/event-subscription-create.png)

    | 設定      | 建議的值  | 說明                                        |
    | ------------ |  ------- | -------------------------------------------------- |
    | **名稱** | imageresizersub | 用以識別新事件訂閱的名稱。 | 
    | **主題類型** |  儲存體帳戶 | 選擇儲存體帳戶事件提供者。 | 
    | **訂用帳戶** | 您的 Azure 訂用帳戶 | 預設會選取您目前的 Azure 訂用帳戶。   |
    | **資源群組** | myResourceGroup | 選取 [使用現有]  ，並選擇您在本教學課程中一直使用的資源群組。  |
    | **Resource** |  您的 Blob 儲存體帳戶 |  選擇您建立的 Blob 儲存體帳戶。 |
    | **事件類型** | 已建立 Blob | 取消勾選 [已建立 Blob]  以外的所有類型。 只有 `Microsoft.Storage.BlobCreated` 的事件類型會傳遞至函式。| 
    | **訂閱者類型** |  自動產生 | FunctionApp |
    | **訂閱者端點** | 自動產生 | Thumbnail。 | 
4. 切換至 [篩選條件]  索引標籤，執行下列動作：     
    1. 選取 [啟用主旨篩選]  選項。
    2. 針對 [主旨開頭]  ，輸入下列值： **/blobServices/default/containers/images/blobs/** 。

        ![指定事件訂閱的篩選條件](https://docs.microsoft.com/zh-tw/azure/event-grid/media/resize-images-on-storage-blob-upload-event/event-subscription-filter.png) 
2. 選取 [建立]  以新增事件訂閱。 當 Blob 新增至 `images` 容器時，這會建立可觸發 `Thumbnail` 函式的事件訂閱。 此函式會調整映像大小，並將其新增至 `thumbnails` 容器。

既然已設定了後端服務，您可以在範例 Web 應用程式中測試映像調整大小功能。 

## <a name="test-the-sample-app"></a>測試範例應用程式

若要在 Web 應用程式中測試映像調整大小，請瀏覽至已發佈應用程式的 URL。 Web 應用程式的預設 URL 是 `https://<web_app>.azurewebsites.net`。

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)

按一下 [上傳相片]  區域，以選取並上傳檔案。 您也可以將相片拖曳到此區域。 

請注意，上傳的映像消失之後，上傳映像的複本會顯示在 [產生縮圖]  浮動切換中。 此映像已由函式調整大小、新增至 *thumbnails* 容器，並由 Web 用戶端下載。

![瀏覽器中已發佈的 Web 應用程式]https://docs.microsoft.com/zh-tw/azure/event-grid/media/resize-images-on-storage-blob-upload-event/tutorial-completed.png)

---
