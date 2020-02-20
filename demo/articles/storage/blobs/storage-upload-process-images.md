
# <a name="tutorial-upload-image-data-in-the-cloud-with-azure-storage"></a>教學課程：使用 Azure 儲存體在雲端中上傳影像資料

本教學課程是一個系列的第一部分。 本教學課程會示範如何部署 Web 應用程式，其使用 Azure 儲存體用戶端程式庫將映像上傳至儲存體帳戶。 完成後，您就有可儲存和顯示 Azure 儲存體映像資料的 Web 應用程式。

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)
![.NET 中的影像大小調整器應用程式](https://docs.microsoft.com/zh-tw/azure/storage/blobs/media/storage-upload-process-images/figure2.png)

---

在系列的第一部分中，您將了解如何：

> * 建立儲存體帳戶
> * 建立容器並設定權限
> * 擷取存取金鑰
> * 將 Web 應用程式部署至 Azure
> * 進行應用程式設定
> * 與 Web 應用程式互動

## <a name="prerequisites"></a>必要條件

若要完成此教學課程，您需要 Azure 訂用帳戶。 開始之前，請建立[免費帳戶](https://azure.microsoft.com/free/?ref=microsoft.com&utm_source=microsoft.com&utm_medium=docs&utm_campaign=visualstudio)。

## <a name="create-a-resource-group"></a>建立資源群組 

使用 [az group create](/cli/azure/group) 命令來建立資源群組。 Azure 資源群組是在其中部署與管理 Azure 資源的邏輯容器。  

下列範例會建立名為 `myResourceGroup` 的資源群組。

```azurecli-interactive
az group create --name myResourceGroup --location eastus

```

## <a name="create-a-storage-account"></a>建立儲存體帳戶

此範例會將影像上傳至 Azure 儲存體帳戶的 Blob 容器。 儲存體帳戶提供唯一命名空間來儲存及存取您的 Azure 儲存體資料物件。 在使用 [az storage account create](/cli/azure/storage/account) 命令所建立的資源群組中建立儲存體帳戶。


在下列命令中，使用您自己的全域唯一名稱來取代見到 `<blob_storage_account>` 預留位置的 Blob 儲存體帳戶。

```azurecli-interactive
blobStorageAccount="<blob_storage_account>"

az storage account create --name $blobStorageAccount --location eastus --resource-group myResourceGroup --sku Standard_LRS --kind blobstorage --access-tier hot

```

## <a name="create-blob-storage-containers"></a>建立 Blob 儲存體容器

應用程式在 Blob 儲存體帳戶中使用兩個容器。 容器類似資料夾，可儲存 Blob。 應用程式會將高解析度的影像上傳到 [影像]  容器。 在系列的後半部分，Azure 函式應用程式會將已調整大小的影像縮圖上傳到 [縮圖]  容器。

使用 [az storage account keys list](/cli/azure/storage/account/keys) 命令取得儲存體帳戶金鑰。 接著，使用此金鑰透過 [az storage container create](/cli/azure/storage/container) 命令來建立兩個容器。  

images  容器的公用存取設為 `off`。 thumbnails  容器的公用存取設為 `container`。 `container` 公用存取設定允許使用者瀏覽網頁來檢視縮圖。

```azurecli-interactive
blobStorageAccountKey=$(az storage account keys list -g myResourceGroup -n $blobStorageAccount --query [0].value --output tsv)

az storage container create -n images --account-name $blobStorageAccount --account-key $blobStorageAccountKey --public-access off

az storage container create -n thumbnails --account-name $blobStorageAccount --account-key $blobStorageAccountKey --public-access container

echo "Make a note of your Blob storage account key..."
echo $blobStorageAccountKey

```

請記下您的 Blob 儲存體帳戶名稱和金鑰。 範例應用程式會使用下列設定連線到儲存體帳戶，以上傳映像。 

## <a name="create-an-app-service-plan"></a>建立應用程式服務方案


使用 [az appservice plan create](/cli/azure/appservice/plan) 命令來建立 App Service 方案。

下列範例會在**免費**定價層中建立名為 `myAppServicePlan` 的 App Service 方案。

```azurecli-interactive
az appservice plan create --name myAppServicePlan --resource-group myResourceGroup --sku Free

```

## <a name="create-a-web-app"></a>建立 Web 應用程式

Web 應用程式提供裝載範例應用程式程式碼的空間，此程式碼是從 GitHub 範例存放庫部署。 使用 [az webapp create](/cli/azure/webapp) 命令，在 `myAppServicePlan` App Service 方案中建立 Web 應用程式

在下列命令中，使用唯一的名稱取代 `<web_app>`。 有效字元是 `a-z`、`0-9` 和 `-`。 如果 `<web_app>` 不是唯一的，您會收到錯誤訊息：_具有指定名稱 `<web_app>` 的網站已經存在。_ Web 應用程式的預設 URL 是 `https://<web_app>.azurewebsites.net`。  

```azurecli-interactive
webapp="<web_app>"

az webapp create --name $webapp --resource-group myResourceGroup --plan myAppServicePlan

```

## <a name="deploy-the-sample-app-from-the-github-repository"></a>從 GitHub 存放庫部署範例應用程式

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)

應用程式服務支援數種將內容部署至 Web 應用程式的方法。 在本教學課程中，您會從[公用 GitHub 範例存放庫](https://github.com/Azure-Samples/storage-blob-upload-from-webapp)部署 Web 應用程式。 使用 [az webapp deployment source config](/cli/azure/webapp/deployment/source) 命令設定 Web 應用程式的 GitHub 部署。

範例專案包含 [ASP.NET MVC](https://www.asp.net/mvc) 應用程式。 此應用程式可接受映像、將它儲存到儲存體帳戶，顯示縮圖容器中的映像。 Web 應用程式使用來自 Azure 儲存體用戶端程式庫的 [Microsoft.Azure.Storage](/dotnet/api/overview/azure/storage)、[Microsoft.Azure.Storage.Blob](/dotnet/api/microsoft.azure.storage.blob) 和 Microsoft.Azure.Storage.Auth 命名空間與 Azure 儲存體進行互動。

```azurecli-interactive
az webapp deployment source config --name $webapp --resource-group myResourceGroup --branch master --manual-integration --repo-url https://github.com/Azure-Samples/storage-blob-upload-from-webapp

```

---

## <a name="configure-web-app-settings"></a>設定 Web 應用程式設定

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)

範例 Web 應用程式使用 [Azure 儲存體用戶端程式庫](/dotnet/api/overview/azure/storage)要求存取權杖，其用來上傳影像。 儲存體 SDK 所使用的儲存體帳戶認證是在 Web 應用程式的應用程式設定中設定。 使用 [az webapp config appsettings set](/cli/azure/webapp/config/appsettings) 命令將應用程式設定新增至已部署的應用程式。

```azurecli-interactive
az webapp config appsettings set --name $webapp --resource-group myResourceGroup --settings AzureStorageConfig__AccountName=$blobStorageAccount AzureStorageConfig__ImageContainer=images AzureStorageConfig__ThumbnailContainer=thumbnails AzureStorageConfig__AccountKey=$blobStorageAccountKey

```

---

部署及設定 Web 應用程式之後，您可以在應用程式中測試映像上傳功能。

## <a name="upload-an-image"></a>上傳影像

若要測試 Web 應用程式，請瀏覽至已發佈應用程式的 URL。 Web 應用程式的預設 URL 是 `https://<web_app>.azurewebsites.net`。

# <a name="nettabdotnet"></a>[\.NET](#tab/dotnet)

選取 [上傳相片]  區域以選取並上傳檔案，或將檔案拖曳到區域。 如果上傳成功，影像就會消失。 [產生的縮圖]  區段就會維持空白，直到我們在本主題稍後對其完成測試。

![在 .NET 中上傳相片](https://docs.microsoft.com/zh-tw/azure/storage/blobs/media/storage-upload-process-images/figure1.png)

在範例程式碼中，Storagehelper.cs  檔案中的 `UploadFiletoStorage` 工作，可供使用 [UploadFromStreamAsync](/dotnet/api/microsoft.azure.storage.blob.cloudblockblob.uploadfromstreamasync) 方法將映像上傳至儲存體帳戶內的 images  容器。 下列程式碼範例包含 `UploadFiletoStorage` 工作。

```csharp
public static async Task<bool> UploadFileToStorage(Stream fileStream, string fileName, AzureStorageConfig _storageConfig)
{
    // Create storagecredentials object by reading the values from the configuration (appsettings.json)
    StorageCredentials storageCredentials = new StorageCredentials(_storageConfig.AccountName, _storageConfig.AccountKey);

    // Create cloudstorage account by passing the storagecredentials
    CloudStorageAccount storageAccount = new CloudStorageAccount(storageCredentials, true);

    // Create the blob client.
    CloudBlobClient blobClient = storageAccount.CreateCloudBlobClient();

    // Get reference to the blob container by passing the name by reading the value from the configuration (appsettings.json)
    CloudBlobContainer container = blobClient.GetContainerReference(_storageConfig.ImageContainer);

    // Get the reference to the block blob from the container
    CloudBlockBlob blockBlob = container.GetBlockBlobReference(fileName);

    // Upload the file
    await blockBlob.UploadFromStreamAsync(fileStream);

    return await Task.FromResult(true);
}
```

以下是前面工作中使用的類別和方法：

|類別  |方法  |
|---------|---------|
|[StorageCredentials](/dotnet/api/microsoft.azure.cosmos.table.storagecredentials)     |         |
|[CloudStorageAccount](/dotnet/api/microsoft.azure.cosmos.table.cloudstorageaccount)    |  [CreateCloudBlobClient](/dotnet/api/microsoft.azure.storage.blob.blobaccountextensions.createcloudblobclient)       |
|[CloudBlobClient](/dotnet/api/microsoft.azure.storage.blob.cloudblobclient)     |[GetContainerReference](/dotnet/api/microsoft.azure.storage.blob.cloudblobclient.getcontainerreference)         |
|[CloudBlobContainer](/dotnet/api/microsoft.azure.storage.blob.cloudblobcontainer)    | [GetBlockBlobReference](/dotnet/api/microsoft.azure.storage.blob.cloudblobcontainer.getblockblobreference)        |
|[CloudBlockBlob](/dotnet/api/microsoft.azure.storage.blob.cloudblockblob)     | [UploadFromStreamAsync](/dotnet/api/microsoft.azure.storage.file.cloudfile.uploadfromstreamasync)        |


選取 [選擇檔案]  來選取檔案，然後按一下 [上傳映像]  。 [產生的縮圖]  區段就會維持空白，直到我們在本主題稍後對其完成測試。 


## <a name="verify-the-image-is-shown-in-the-storage-account"></a>確定影像顯示在儲存體帳戶中

登入 [Azure 入口網站](https://portal.azure.com)。 從左側的功能表中選取 [儲存體帳戶]  ，然後選取您的儲存體帳戶名稱。 選取 [容器]  ，然後選取**映像**容器。

確認影像顯示在容器中。

![Azure 入口網站的影像清單容器](https://docs.microsoft.com/zh-tw/azure/storage/blobs/media/storage-upload-process-images/figure13.png)

