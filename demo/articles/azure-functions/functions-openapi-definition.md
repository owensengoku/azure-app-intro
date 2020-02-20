
# <a name="create-an-openapi-definition-for-a-serverless-api-using-azure-api-management"></a>使用 Azure API 管理為無伺服器 API 建立 OpenAPI 定義

REST API 通常會使用 OpenAPI 定義來描述。 此定義包含有關 API 中可以使用哪些作業，以及應該如何結構化 API 之要求和回應資料的資訊。

在本教學課程中，您會建立一個函式，來判斷風力渦輪機的緊急修復是否符合成本效益。 然後，您會使用 Azure API 管理 為函式應用程式建立 OpenAPI 定義，以便從其他應用程式和服務呼叫函式。

![Azure API 管理概念圖](https://docs.microsoft.com/en-us/azure/api-management/media/api-management-using-with-vnet/api-management-vnet-external.png)

在本教學課程中，您會了解如何：

> * 在 Azure 中建立函式
> * 使用 Azure API 管理產生 OpenAPI 定義
> * 呼叫函式以測試定義
> * 下載 OpenAPI 定義

## <a name="create-a-function-app"></a>建立函數應用程式

您必須擁有函式應用程式以便主控函式的執行。 函式應用程式可讓您將多個函式群組為邏輯單位，以方便您管理、部署、調整和共用資源。

1. 從 Azure 入口網站功能表選取 [建立資源]  。

    ![使用 Azure 入口網站功能表新增資源](https://docs.microsoft.com/zh-tw/azure/includes/media/functions-create-function-app-portal/create-function-app-resource.png)

1. 在 [新增]  頁面中，選取 [計算]   > [函數應用程式]  。

1. 請使用影像下面的資料表中指定的函式應用程式設定。

    ![基本概念](https://docs.microsoft.com/zh-tw/azure/includes/media/functions-create-function-app-portal/function-app-create-basics.png)

    | 設定      | 建議的值  | 描述 |
    | ------------ | ---------------- | ----------- |
    | **訂用帳戶** | 您的訂用帳戶 | 將在其下建立這個新函式應用程式的訂用帳戶。 |
    | **[資源群組]** |  *myResourceGroup* | 要在其中建立函式應用程式的新資源群組名稱。 |
    | **函式應用程式名稱** | 全域唯一的名稱 | 用以識別新函式應用程式的名稱。 有效字元為 `a-z` (區分大小寫)、`0-9` 以及 `-`。  |
    |**Publish**| 程式碼 | 發佈程式碼檔案或 Docker 容器的選項。 |
    | **執行階段堆疊** | 慣用語言 | 選擇支援您慣用函式程式設計語言的執行階段。 針對 C# 和 F # 函式選擇 **.NET**。 |
    |**區域**| 慣用區域 | 美國東部 |

    選取 [下一步:  裝載 >] 按鈕。

1. 輸入下列設定以進行裝載。

    ![裝載](https://docs.microsoft.com/zh-tw/azure/includes/media/functions-create-function-app-portal/function-app-create-hosting.png)

    | 設定      | 建議的值  | 描述 |
    | ------------ | ---------------- | ----------- |
    | **[儲存體帳戶](../articles/storage/common/storage-account-create.md)** |  全域唯一的名稱 |  建立您函式應用程式使用的儲存體帳戶。 儲存體帳戶名稱必須介於 3 到 24 個字元的長度，而且只能包含數字和小寫字母。 您也可以使用現有帳戶，條件是必須符合[儲存體帳戶需求](../articles/azure-functions/functions-scale.md#storage-account-requirements)。 |
    |**作業系統**| 慣用的作業系統 | 系統會根據您的執行階段堆疊選項預先選取作業系統，但您可以視需要變更設定。 |
    | **[規劃](../articles/azure-functions/functions-scale.md)** | 取用方案 | 會定義如何將資源配置給函式應用程式的主控方案。 在預設**取用方案**中，您的函式會根據需要來動態新增資源。 在此[無伺服器](https://azure.microsoft.com/overview/serverless-computing/)裝載中，您只需要針對函式有執行的時間來付費。 在 App Service 方案中執行時，您必須管理[函式應用程式的調整](../articles/azure-functions/functions-scale.md)。  |

    選取 [下一步:  監視 >] 按鈕。

1. 輸入下列設定以進行監視。

    ![監視](https://docs.microsoft.com/zh-tw/azure/includes/media/functions-create-function-app-portal/function-app-create-monitoring.png)

    | 設定      | 建議的值  | 描述 |
    | ------------ | ---------------- | ----------- |
    | **[Application Insights](../articles/azure-functions/functions-monitoring.md)** | 預設 | 在最近的支援區域中，建立相同*應用程式名稱*的 Application Insights 資源。 您可以展開此設定，變更 [新資源名稱]  或在 [Azure 地理位置](https://azure.microsoft.com/global-infrastructure/geographies/)中依您希望儲存資料的地點，選擇不同的**位置**。 |

    選取 [檢閱 + 建立]  ，以檢閱應用程式組態選項。

1. 選取 [建立]  以佈建並部署函式應用程式。

1. 選取入口網站右上角的 [通知] 圖示，查看是否有**部署成功**訊息。

    ![部署通知](https://docs.microsoft.com/zh-tw/azure/includes/media/functions-create-function-app-portal/function-app-create-notification2.png)

1. 選取 [前往資源]  ，以檢視您新的函式應用程式。 您也可以選取 [釘選到儀表板]  。 釘選可讓您更輕鬆地從儀表板返回此函式應用程式資源。


## <a name="create-the-function"></a>建立函式

本教學課程會使用 HTTP 觸發的函式，並採用兩個參數：

* 修復渦輪機的預估時間 (以小時為單位)。
* 渦輪機的容量 (以千瓦為單位)。 

然後，此函式會計算修復的費用，以及渦輪機在 24 小時內的收入。 若要在 [Azure 入口網站](https://portal.azure.com)中建立 HTTP 觸發的函式：

1. 展開函式應用程式，然後選取 [函式]  旁的 [+]  按鈕。 選取 [入口網站內]   > [繼續]  。

1. 選取 [更多範本...]  ，然後選取 [完成並檢視範本] 

1. 選取 HTTP 觸發程序，輸入 `TurbineRepair` 作為函式 [名稱]  ，選擇 `Function` 作為 **[[驗證層級]](functions-bindings-http-webhook.md#http-auth)** ，然後選取 [建立]  。  

    ![建立適用於 OpenAPI 的 HTTP 函式](https://docs.microsoft.com/zh-tw/azure/azure-functions/media/functions-openapi-definition/select-http-trigger-openapi.png)

1. 將 run.csx C# 指令碼檔案的內容取代為下列程式碼，然後選擇 [儲存]  ：

    ```csharp
    #r "Newtonsoft.Json"
    
    using System.Net;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Extensions.Primitives;
    using Newtonsoft.Json;
    
    const double revenuePerkW = 0.12;
    const double technicianCost = 250;
    const double turbineCost = 100;
    
    public static async Task<IActionResult> Run(HttpRequest req, ILogger log)
    {
        // Get query strings if they exist
        int tempVal;
        int? hours = Int32.TryParse(req.Query["hours"], out tempVal) ? tempVal : (int?)null;
        int? capacity = Int32.TryParse(req.Query["capacity"], out tempVal) ? tempVal : (int?)null;
    
        // Get request body
        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        dynamic data = JsonConvert.DeserializeObject(requestBody);
    
        // Use request body if a query was not sent
        capacity = capacity ?? data?.capacity;
        hours = hours ?? data?.hours;
    
        // Return bad request if capacity or hours are not passed in
        if (capacity == null || hours == null){
            return new BadRequestObjectResult("Please pass capacity and hours on the query string or in the request body");
        }
        // Formulas to calculate revenue and cost
        double? revenueOpportunity = capacity * revenuePerkW * 24;  
        double? costToFix = (hours * technicianCost) +  turbineCost;
        string repairTurbine;
    
        if (revenueOpportunity > costToFix){
            repairTurbine = "Yes";
        }
        else {
            repairTurbine = "No";
        };
    
        return (ActionResult)new OkObjectResult(new{
            message = repairTurbine,
            revenueOpportunity = "$"+ revenueOpportunity,
            costToFix = "$"+ costToFix
        });
    }
    ```

    此函式程式碼會傳回 `Yes` 或 `No` 的訊息，指出緊急修復是否符合成本效益，以及渦輪機所代表的收入機會與修復渦輪機的成本。

1. 若要測試函式，請按一下最右邊的 [測試]  ，將 [測試] 索引標籤展開。針對 [要求本文]  輸入下列值，然後按一下 [執行]  。

    ```json
    {
    "hours": "6",
    "capacity": "2500"
    }
    ```

    ![在 Azure 入口網站中測試函式](https://docs.microsoft.com/zh-tw/azure/azure-functions/media/functions-openapi-definition/test-function.png)

    下列值會在回應本文中傳回。

    ```json
    {"message":"Yes","revenueOpportunity":"$7200","costToFix":"$1600"}
    ```

現在，您有一個函式，可判斷緊急修復是否符合成本效益。 接下來，您會為函式應用程式產生 OpenAPI 定義。

## <a name="generate-the-openapi-definition"></a>產生 OpenAPI 定義

您現在已經準備好產生 OpenAPI 定義。

1. 選取函數應用程式，然後在 [平台功能]  中，選擇 [API 管理]  ，然後選取 [API 管理]  下的 [新建]  。

    ![在 [平台功能] 中選擇 [API 管理]](https://docs.microsoft.com/zh-tw/azure/azure-functions/media/functions-openapi-definition/select-all-settings-openapi.png)

1. 使用影像下方的資料表中指定的 API 管理設定。

    ![建立新的 API 管理服務](https://docs.microsoft.com/zh-tw/azure/azure-functions/media/functions-openapi-definition/new-apim-service-openapi.png)

    | 設定      | 建議的值  | 描述                                        |
    | ------------ |  ------- | -------------------------------------------------- |
    | **名稱** | 全域唯一的名稱 | 系統會根據您的函式應用程式名稱產生名稱。 |
    | **訂用帳戶** | 您的訂用帳戶 | 這項新資源建立所在的訂用帳戶。 |  
    | **[資源群組** |  myResourceGroup | 與您的函式應用程式相同的資源，系統應該會為您設定。 |
    | **位置** | 美國東部 | 選擇 [美國東部] 位置。 |
    | **組織名稱** | Contoso | 用於開發人員入口網站和電子郵件通知的組織名稱。 |
    | **管理員電子郵件** | 您的電子郵件 | 從 API 管理接收系統通知的電子郵件。 |
    | **定價層** | 使用量 (預覽) | 取用量層目前為預覽狀態，且不是所有地區都能使用。 如需完整的定價詳細資料，請參閱 [API 管理定價頁面](https://azure.microsoft.com/pricing/details/api-management/) |

1. 選擇 [建立]  以建立 API 管理執行個體，這可能需要幾分鐘的時間。

1. 選取 [啟用 Application Insights]  以將記錄傳送至與函式應用程式相同的位置，然後接受其餘的預設值，並選取 [連結 API]  。

1. [匯入 Azure Functions]  隨即開啟，並醒目提示 **TurbineRepair** 函式。 選擇 [選取]  以繼續操作。

    ![將 Azure Functions 匯入 API 管理中](https://docs.microsoft.com/zh-tw/azure/azure-functions/media/functions-openapi-definition/import-function-openapi.png)

1. 在 [從函式應用程式建立]  頁面上接受預設值，然後選取 [建立] 

    ![從函式應用程式建立](https://docs.microsoft.com/zh-tw/azure/azure-functions/media/functions-openapi-definition/create-function-openapi.png)

現在已建立函式的 API。

## <a name="test-the-api"></a>測試 API

使用 OpenAPI 定義之前，應先確認 API 能夠運作。

1. 在函式的 [測試]  索引標籤中，選取 [POST]  作業。

1. 輸入 [時數]  和 [容量]  的值

    ```json
    {
    "hours": "6",
    "capacity": "2500"
    }
    ```

1. 按一下 [傳送]  ，然後檢視 HTTP 回應。

    ![測試函式 API](https://docs.microsoft.com/zh-tw/azure/azure-functions/media/functions-openapi-definition/test-function-api-openapi.png)

## <a name="download-the-openapi-definition"></a>下載 OpenAPI 定義

如果您的 API 可以正常運作，就可以下載 OpenAPI 定義。

1. 選取頁面頂端的 [下載 OpenAPI 定義]  。
   
   ![下載 OpenAPI 定義](https://docs.microsoft.com/zh-tw/azure/azure-functions/media/functions-openapi-definition/download-definition.png)

2. 開啟下載的 JSON 檔案並檢閱定義。


