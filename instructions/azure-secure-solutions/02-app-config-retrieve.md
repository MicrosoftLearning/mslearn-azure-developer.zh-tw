---
lab:
  topic: Secure solutions in Azure
  title: 從 Azure 應用程式組態擷取組態設定
  description: 了解如何建立 Azure 應用程式組態資源，以及使用 Azure CLI 設定組態資訊。 然後，使用 **ConfigurationBuilder** 擷取應用程式的設定。
---

# 從 Azure 應用程式組態擷取組態設定

在本練習中，您將建立 Azure 應用程式組態資源、使用 Azure CLI 儲存組態設定，以及建置使用 **ConfigurationBuilder** 擷取設定值的 .NET 主控台應用程式。 您將了解如何使用階層式索引鍵來組織設定，並驗證您的應用程式以存取雲端式設定資料。

在此練習中執行的工作：

* 建立 Azure 應用程式組態資源
* 儲存連接字串設定資訊
* 建立 .NET 主控台應用程式以擷取設定資訊
* 清除資源

本練習大約需要 **15** 分鐘才能完成。

## 建立 Azure 應用程式組態資源並新增設定資訊

在本練習的此章節中，您將使用 Azure CLI 在 Azure 中建立所需的資源。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。

1. 使用頁面上方搜尋欄右側的 **[\>_]** 按鈕，就能從 Azure 入口網站建立新的 Cloud Shell，並選取 ***Bash*** 環境。 Cloud Shell 會在 Azure 入口網站底部的窗格顯示命令列介面。 如果系統提示您選取儲存體帳戶以保存檔案，請選取 [不需要儲存體帳戶]****、[您的訂用帳戶]，然後選取 [套用]****。

    > **備註**：如果您之前就已建立使用 *Bash* 環境的 Cloud Shell，請將原先的設定切換成 ***PowerShell***。

1. 在 Cloud Shell 工具列中，在**設定**功能表中，選擇**轉到經典版本**（這是使用程式碼編輯器所必需的）。

1. 針對本練習所需的資源建立資源群組。 如果您已有想要使用的資源群組，請繼續進行下一個步驟。 將 **myResourceGroup** 取代為您想要用於資源群組的名稱。 如有必要，您可以使用附近的地區來取代 **eastus**。

    ```
    az group create --name myResourceGroup --location eastus
    ```

1. 許多命令需要唯一名稱，並使用相同的參數。 建立一些變數將減少建立資源的命令所需的變更。 執行下列命令來建立所需的變數。 將 **myResourceGroup** 取代為您用於此練習的名稱。 如果您在上一個步驟中變更了位置，請在**位置**變數中進行相同的變更。

    ```
    resourceGroup=myResourceGroup
    location=eastus
    appConfigName=appconfigname$RANDOM
    ```

1. 執行下列命令以取得應用程式組態資源的名稱。 記錄名稱，該名稱將在稍後於本練習中使用。

    ```
    echo $appConfigName
    ```

1. 執行下列命令，確保 **Microsoft.AppConfiguration** 提供者已在您的訂閱中註冊。

    ```
    az provider register --namespace Microsoft.AppConfiguration
    ```

1. 這可能需要幾分鐘的時間才能完成註冊。 執行下列命令，檢查註冊狀態。 當結果傳回**已註冊**時，繼續進行下一個步驟。

    ```
    az provider show --namespace Microsoft.AppConfiguration --query "registrationState"
    ```

1. 執行下列命令以建立 Azure 應用程式組態資源。 執行可能需要數分鐘的時間。

    ```
    az appconfig create --location $location \
        --name $appConfigName \
        --resource-group $resourceGroup
        --sku Free
    ```

    >**提示：** 若由於使用**免費** SKU 值的配額限制，導致建立 AppConfig 資源時發生問題，請改用**開發人員**。
    

### 將角色指派給 Microsoft Entra 使用者名稱

若要擷取設定資訊，您必須將 Microsoft Entra 使用者指派給**應用程式組態資料讀取者**角色。 

1. 執行下列命令，從您的帳戶中擷取 **userPrincipalName**。 這代表角色將指派給誰。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 執行下列命令，以擷取應用程式組態服務的資源識別碼。 資源識別碼會設定角色指派的範圍。

    ```
    resourceID=$(az appconfig show --resource-group $resourceGroup \
        --name $appConfigName --query id --output tsv)
    ```

1. 執行下列命令以建立並指派**應用程式組態資料讀取器**角色。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "App Configuration Data Reader" \
        --scope $resourceID
    ```

接下來，將預留位置連接字串新增至應用程式組態。

### 使用 Azure CLI 新增組態資訊

在 Azure 應用程式組態中，如 **Dev：conStr** 的索引鍵是階層式索引鍵或命名空間索引鍵。 冒號 (:) 可作為建立邏輯階層的分隔符號，其中：

* **Dev** 代表命名空間或環境前置詞 (表示此設定適用於開發環境)
* **conStr** 代表設定名稱

這種階層式結構可讓您依環境、功能或應用程式元件組織組態設定，從而更輕鬆地管理和擷取相關設定。

執行下列命令以儲存預留位置連接字串。 

```
az appconfig kv set --name $appConfigName \
    --key Dev:conStr \
    --value connectionString \
    --yes
```

此命令會傳回一些 JSON。 最後一行包含純文字值。 

```json
"value": "connectionString"
```

## 建立 .NET 主控台應用程式以擷取設定資訊

現在所需的資源已部署至 Azure，下一個步驟是設定主控台應用程式。 下列步驟是在 Cloud Shell 中執行。

>**提示：** 拖曳頂端框線調整 Cloud Shell 的大小，以顯示更多資訊和程式碼。 您也可以使用最小化和最大化按鈕，在 Cloud Shell 和主要入口網站介面之間切換。

1. 執行下列指令，以建立包含專案的目錄，並變更為專案目錄。

    ```
    mkdir appconfig
    cd appconfig
    ```

1. 建立 .NET 主控台應用程式。

    ```
    dotnet new console
    ```

1. 執行下列命令，將 **Azure.Identity** 和 **Microsoft.Extensions.Configuration.AzureAppConfiguration** 套件新增至專案。

    ```
    dotnet add package Azure.Identity
    dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
    ```

### 新增專案的程式碼

1. 在 Cloud Shell 中執行下列命令，開始編輯應用程式。

    ```
    code Program.cs
    ```

1. 將任何現有內容取代為下列程式碼。 請務必將 **YOUR_APP_CONFIGURATION_NAME** 取代為您先前記錄的名稱，並閱讀程式碼中的註解。

    ```csharp
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.Configuration.AzureAppConfiguration;
    using Azure.Identity;
    
    // Set the Azure App Configuration endpoint, replace YOUR_APP_CONFIGURATION_NAME
    // with the name of your actual App Configuration service
    
    string endpoint = "https://YOUR_APP_CONFIGURATION_NAME.azconfig.io"; 
    
    // Configure which authentication methods to use
    // DefaultAzureCredential tries multiple auth methods automatically
    DefaultAzureCredentialOptions credentialOptions = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a configuration builder to combine multiple config sources
    var builder = new ConfigurationBuilder();
    
    // Add Azure App Configuration as a source
    // This connects to Azure and loads configuration values
    builder.AddAzureAppConfiguration(options =>
    {
        
        options.Connect(new Uri(endpoint), new DefaultAzureCredential(credentialOptions));
    });
    
    // Build the final configuration object
    try
    {
        var config = builder.Build();
        
        // Retrieve a configuration value by key name
        Console.WriteLine(config["Dev:conStr"]);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error connecting to Azure App Configuration: {ex.Message}");
    }
    ```

1. 按 **ctrl+s** 以儲存檔案，然後按 **ctrl+q** 以結束編輯器。

## 登入 Azure，然後執行應用程式

1. 在 Cloud Shell 中，輸入下列命令以登入 Azure。

    ```
    az login
    ```

    **<font color="red">即使 Cloud Shell 工作階段已經過驗證，您還是必須登錄 Azure。</font>**

    > **注意**：在大部分情況下，只要使用 *az 登入*即可。 不過，如果您在多個租用戶中擁有訂用帳戶，您可能需要使用 *--tenant* 參數指定租用戶。 如需詳細資料，請參閱[使用 Azure CLI 以互動方式登入 Azure](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)。

1. 執行下列命令以啟動主控台應用程式。 應用程式會顯示您稍早在練習中指派給 **Dev：conStr** 設定的 **connectionString** 值。

    ```
    dotnet run
    ```

    應用程式會顯示您稍早在練習中指派給 **Dev：conStr** 設定的 **connectionString** 值。

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。
1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。
