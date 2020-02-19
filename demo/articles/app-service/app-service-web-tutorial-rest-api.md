
# <a name="tutorial-host-a-restful-api-with-cors-in-azure-app-service"></a>教學課程：在 Azure App Service 中裝載具有 CORS 支援的 RESTful API

Azure App Service 可提供可高度擴充、自我修復的 Web 主控服務。 此外，App Service 有 RESTful API 的內建[跨原始資源共用 (CORS)](https://wikipedia.org/wiki/Cross-Origin_Resource_Sharing) 支援。 本教學課程示範如何將 ASP.NET Core API 應用程式部署至具有 CORS 支援的 App Service。 您可使用命令列工具來設定應用程式，以及使用 Git 部署應用程式。 

在本教學課程中，您會了解如何：

> * 使用 Azure CLI 建立 App Service 資源
> * 使用 Git 將 RESTful API 部署到 Azure
> * 啟用 App Service CORS 支援


## <a name="prerequisites"></a>必要條件

若要完成本教學課程：

* [安裝 Git](https://git-scm.com/)。
* [安裝 .NET Core](https://www.microsoft.com/net/core/)。

## <a name="create-local-aspnet-core-app"></a>建立本機 ASP.NET Core 應用程式

在此步驟中，您要設定本機 ASP.NET Core 專案。 App Service 會對以其他語言撰寫的 API 支援相同的工作流程。

### <a name="clone-the-sample-application"></a>複製範例應用程式

在終端機視窗中，使用 `cd` 移至工作目錄。  

執行下列命令來複製範例存放庫。 

```bash
git clone https://github.com/Azure-Samples/dotnet-core-api
```

此存放庫包含根據下列教學課程建立的應用程式：[使用 Swagger 的 ASP.NET Core Web API 說明頁面](/aspnet/core/tutorials/web-api-help-pages-using-swagger?tabs=visual-studio)。 它會使用 Swagger 產生器來提供 [Swagger UI](https://swagger.io/swagger-ui/)和 Swagger JSON 端點。

### <a name="run-the-application"></a>執行應用程式

執行下列命令以安裝必要的套件、執行資料庫移轉，然後啟動應用程式。

```bash
cd dotnet-core-api
dotnet restore
dotnet run
```

在瀏覽器中瀏覽至 `http://localhost:5000/swagger` 以使用 Swagger UI 播放。

![在本機執行的 ASP.NET Core API](https://docs.microsoft.com/zh-tw/azure/app-service/media/app-service-web-tutorial-rest-api/azure-app-service-local-swagger-ui.png)

瀏覽至 `http://localhost:5000/api/todo` 並查看 ToDo JSON 項目清單。

瀏覽至 `http://localhost:5000` 並使用瀏覽器應用程式播放。 稍後，您會將瀏覽器應用程式指向 App Service 中的遠端 API，以測試 CORS 功能。 瀏覽器應用程式的程式碼位於存放庫的 wwwroot  目錄中。

如需隨時停止 ASP.NET Core，請在終端機上按下 `Ctrl+C`。

## <a name="deploy-app-to-azure"></a>將應用程式部署到 Azure

在此步驟中，您會將與 SQL Database 連線的 .NET Core 應用程式部署至 App Service。

### <a name="configure-local-git-deployment"></a>設定本機 git 部署

FTP 和本機 Git 可以透過使用「部署使用者」  部署到 Azure Web 應用程式。 部署使用者只需設定一次，就能供所有 Azure 部署使用。 使用者名稱和密碼屬於帳戶等級部署，因此與 Azure 訂用帳戶認證不同。 

若要設定部署使用者，請在 Azure Cloud Shell 中執行 [az webapp deployment user set](/cli/azure/webapp/deployment/user?view=azure-cli-latest#az-webapp-deployment-user-set) 命令。 以部署使用者的使用者名稱和和密碼來取代「username」\<\<和「password」。 

- 使用者名稱在 Azure 服務及本機 Git 推送中都必須是唯一的，且不能包含 ‘@’ 符號。 
- 密碼長度必須至少為 8 個字元，包含下列三個元素其中兩個：字母、數字及符號。 

```azurecli-interactive
az webapp deployment user set --user-name <username> --password <password>
```

JSON 輸出會將密碼顯示為 `null`。 如果您收到 `'Conflict'. Details: 409` 錯誤，請變更使用者名稱。 如果您收到 `'Bad Request'. Details: 400` 錯誤，請使用更強的密碼。 

將使用者名稱和密碼記錄下來，在部署 Web 應用程式時還會用到。


### <a name="create-a-resource-group"></a>建立資源群組


在 Cloud Shell 中，使用 [`az group create`](/cli/azure/group?view=azure-cli-latest) 命令來建立資源群組。 下列範例會在「西歐」  位置建立名為 myResourceGroup  的資源群組。 若要查看**免類**層中 App Service 的所有支援位置，請執行 [`az appservice list-locations --sku FREE`](/cli/azure/appservice?view=azure-cli-latest) 命令。

```azurecli-interactive
az group create --name myResourceGroup --location "East US"
```

您通常會在附近的區域中建立資源群組和資源。 

當命令完成時，JSON 輸出會顯示資源群組屬性。

### <a name="create-an-app-service-plan"></a>建立應用程式服務方案

下列範例會在**免費**定價層中建立名為 `myAppServicePlan` 的 App Service 方案。

```azurecli-interactive
az appservice plan create --name myAppServicePlan --resource-group myResourceGroup --sku FREE
```

### <a name="create-a-web-app"></a>建立 Web 應用程式

在 `myAppServicePlan` App Service 方案中建立 [Web 應用程式](../articles/app-service/containers/app-service-linux-intro.md)。 

在 Cloud Shell 中，您可以使用 [`az webapp create`](/cli/azure/webapp?view=azure-cli-latest#az-webapp-create) 命令。 在下列範,了中，使用全域唯一的應用程式名稱 (有效的字元為 `a-z`、`0-9` 和 `-`) 取代 `<app-name>`。 

```azurecli-interactive
az webapp create --resource-group myResourceGroup --plan myAppServicePlan --name <app-name> --deployment-local-git
```
輸入指令後會看到類似結果

Local git is configured with url of 'https://devopstaku@myappfordevopstaku.scm.azurewebsites.net/myAppForDevOpstaku.git'

### <a name="push-to-azure-from-git"></a>從 Git 推送至 Azure
回到本機終端視窗，將 Azure 遠端新增至本機 Git 存放庫。 將 *\<deploymentLocalGitUrl-from-create-step>* 取代為您從上一步得到的結果

```bash
git remote add azure <deploymentLocalGitUrl-from-create-step>
```

推送到 Azure 遠端，使用下列命令來部署您的應用程式。 當 Git 認證管理員提示輸入認證時，請務必輸入您在[設定部署使用者](/azure/app-service/containers/tutorial-python-postgresql-app#configure-a-deployment-user)中建立的認證，而不是您用來登入 Azure 入口網站的認證。

```bash
git push azure master
```

此命令可能會花數分鐘執行。 執行上述命令時，會顯示類似下列範例的資訊：


```bash
Counting objects: 98, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (92/92), done.
Writing objects: 100% (98/98), 524.98 KiB | 5.58 MiB/s, done.
Total 98 (delta 8), reused 0 (delta 0)
remote: Updating branch 'master'.
remote: .
remote: Updating submodules.
remote: Preparing deployment for commit id '0c497633b8'.
remote: Generating deployment script.
remote: Project file path: ./DotNetCoreSqlDb.csproj
remote: Generated deployment script files
remote: Running deployment command...
remote: Handling ASP.NET Core Web Application deployment.
remote: .
remote: .
remote: .
remote: Finished successfully.
remote: Running post deployment command(s)...
remote: Deployment successful.
remote: App container will begin restart within 10 seconds.
To https://<app_name>.scm.azurewebsites.net/<app_name>.git
 * [new branch]      master -> master
```

### <a name="browse-to-the-azure-app"></a>瀏覽至 Azure 應用程式

在瀏覽器中瀏覽至 `http://<app_name>.azurewebsites.net/swagger` 並使用 Swagger UI 播放。

![在 Azure App Service 中執行的 ASP.NET Core API](https://docs.microsoft.com/zh-tw/azure/app-service/media/app-service-web-tutorial-rest-api/azure-app-service-browse-app.png)

瀏覽至 `http://<app_name>.azurewebsites.net/swagger/v1/swagger.json` 以查看您已部署 API 的 swagger.json  。

瀏覽至 `http://<app_name>.azurewebsites.net/api/todo` 以查看您已部署的 API 是否能運作。

## <a name="add-cors-functionality"></a>新增 CORS 功能

接著，您會在 App Service 中為 API 啟用內建 CORS 支援。

### <a name="enable-cors"></a>啟用 CORS 

在 Cloud Shell 中，使用 [`az resource update`](/cli/azure/resource#az-resource-update) 命令，對您的用戶端 URL 啟用 CORS。 取代 _&lt;appname>_ 預留位置。

```azurecli-interactive
az resource update --name web --resource-group myResourceGroup --namespace Microsoft.Web --resource-type config --parent sites/<app_name> --set properties.cors.allowedOrigins="*['http://test-domain:5000']*" --api-version 2015-06-01
```

您可以在 `properties.cors.allowedOrigins` (`"['URL1','URL2',...]"`) 中設定多個用戶端 URL。 您也可以使用 `"['*']"` 來啟用所有的用戶端 URL。

> [!NOTE]
> 如果您的應用程式需要傳送認證，例如 Cookie 或驗證權杖，則瀏覽器可能會在回應上要求 `ACCESS-CONTROL-ALLOW-CREDENTIALS` 標頭。 若要在 App Service 中啟用此作業，請在 CORS 組態中將 `properties.cors.supportCredentials` 設定為 `true`。當 `allowedOrigins` 包含 `'*'` 時，無法啟用此作業。


## <a name="app-service-cors-vs-your-cors"></a>App Service CORS 與您的 CORS

您可使用自己的 CORS 公用程式，而不是使用 App Service CORS，以獲得更大彈性。 例如，您可以針對不同的路由或方法，指定不同的允許來源。 因為 App Service CORS 可讓您針對所有的 API 路由和方法指定一組可接受的來源，所以您可使用自己的 CORS 程式碼。 在[啟用跨源要求 (CORS)](/aspnet/core/security/cors)中了解 ASP.NET Core 如何執行此作業。

> [!NOTE]
> 請勿嘗試同時使用 App Service CORS 與您自己的 CORS 程式碼。 一起使用時，App Service CORS 具有優先權，而您自己的 CORS 程式碼沒有任何作用。
>
>
