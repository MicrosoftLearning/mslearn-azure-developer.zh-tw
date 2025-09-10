---
lab:
  topic: Azure events and messaging
  title: 使用 Azure 事件方格將事件路由至自訂端點
  description: 了解如何使用 Azure 事件方格將事件路由至自訂端點。
---

# 使用 Azure 事件方格將事件路由至自訂端點

在本練習中，您將建立 Azure 事件方格主題和 Web 應用程式端點，然後建置 .NET 主控台應用程式，將自訂事件傳送至事件方格主題。 您將了解如何設定事件訂閱、使用事件方格進行驗證，以及透過在 Web 應用程式中檢視事件，確認事件已成功路由傳送至端點。

在此練習中執行的工作：

* 建立 Azure 事件方格資源
* 啟用事件方格資源提供者
* 在事件方格中建立主題
* 建立訊息端點
* 訂閱主題
* 使用 .NET 主控台應用程式傳送事件
* 清除資源

本練習大約需要 **30** 分鐘才能完成。

## 建立 Azure 事件方格資源

在本練習的此章節中，您將使用 Azure CLI 在 Azure 中建立所需的資源。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。

1. 使用頁面上方搜尋欄右側的 **[\>_]** 按鈕，就能從 Azure 入口網站建立新的 Cloud Shell，並選取 ***Bash*** 環境。 Cloud Shell 會在 Azure 入口網站底部的窗格顯示命令列介面。 如果系統提示您選取儲存體帳戶以保存檔案，請選取 [不需要儲存體帳戶]****、[您的訂用帳戶]，然後選取 [套用]****。

    > **備註**：如果您之前就已建立使用 *Bash* 環境的 Cloud Shell，請將原先的設定切換成 ***PowerShell***。

1. 在 Cloud Shell 工具列中，在**設定**功能表中，選擇**轉到經典版本**（這是使用程式碼編輯器所必需的）。

1. 針對本練習所需的資源建立資源群組。 如果您已有想要使用的資源群組，請繼續進行下一個步驟。 將 **myResourceGroup** 取代為您想要用於資源群組的名稱。 如有必要，您可以使用附近的地區來取代 **eastus**。

    ```bash
    az group create --name myResourceGroup --location eastus
    ```

1. 許多命令需要唯一名稱，並使用相同的參數。 建立一些變數將減少建立資源的命令所需的變更。 執行下列命令來建立所需的變數。 將 **myResourceGroup** 取代為您用於此練習的名稱。 如果您在上一個步驟中變更了位置，請在**位置**變數中進行相同的變更。

    ```bash
    let rNum=$RANDOM
    resourceGroup=myResourceGroup
    location=eastus
    topicName="mytopic-evgtopic-${rNum}"
    siteName="evgsite-${rNum}"
    siteURL="https://${siteName}.azurewebsites.net"
    ```

### 啟用事件方格資源提供者

Azure 資源提供者是定義和管理 Azure 中特定資源類型的服務。 當您部署或管理資源時，這便是 Azure 在幕後使用的資訊。 使用 **az provider register** 命令註冊事件方格資源提供者。 

```bash
az provider register --namespace Microsoft.EventGrid
```

這可能需要幾分鐘的時間才能完成註冊。 您可以使用下列命令檢查狀態。

```bash
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```

> **注意：** 此步驟只有先前未使用事件方格的訂閱才需要。

### 在事件方格中建立主題

使用 **az eventgrid topic create** 命令來建立主題。 名稱必須為唯一，因為其為 DNS 項目的一部分。  

```bash
az eventgrid topic create --name $topicName \
    --location $location \
    --resource-group $resourceGroup
```

### 建立訊息端點

訂閱自訂主題之前，必須先建立事件訊息的端點。 端點通常會根據事件資料來採取動作。 下列指令碼使用預先建置的 Web 應用程式來顯示事件訊息。 部署的解決方案包括 App Service 方案、App Service Web 應用程式，以及 GitHub 的原始程式碼。

1. 執行下列命令以建立訊息端點。 **echo** 命令將顯示端點的網站 URL。

    ```bash
    az deployment group create \
        --resource-group $resourceGroup \
        --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/main/azuredeploy.json" \
        --parameters siteName=$siteName hostingPlanName=viewerhost
    
    echo "Your web app URL: ${siteURL}"
    ```

    > **注意：** 此命令可能會需要花費幾分鐘才能完成。

1. 在瀏覽器中開啟新索引標籤，瀏覽至上述指令碼結尾所產生的 URL，以確保 Web 應用程式正在執行。 您應會看到網站目前未顯示任何訊息。

    > **提示：** 讓瀏覽器保持執行狀態，因為其用於顯示更新。

### 訂閱主題

您會訂閱事件方格主題，以將您要追蹤的事件與要傳送那些事件的位置告訴事件方線。 

1. 使用 **az eventgrid event-subscription create** 命令訂閱主題。 下列指令碼會從您的帳戶擷取訂閱識別碼，並在建立事件訂閱時使用。

    ```bash
    endpoint="${siteURL}/api/updates"
    topicId=$(az eventgrid topic show --resource-group $resourceGroup \
        --name $topicName --query "id" --output tsv)
    
    az eventgrid event-subscription create \
        --source-resource-id $topicId \
        --name TopicSubscription \
        --endpoint $endpoint
    ```

1. 再次檢視您的 Web 應用程式，並注意訂閱驗證事件是否已傳送至其中。 選取眼睛圖示以展開事件資料。 事件方格會傳送驗證事件，以便端點確認接收事件資料。 Web 應用程式包括用於驗證訂閱的程式碼。

## 使用 .NET 主控台應用程式傳送事件

現在所需的資源已部署至 Azure，下一個步驟是設定主控台應用程式。 下列步驟是在 Cloud Shell 中執行。

>**提示：** 拖曳頂端框線調整 Cloud Shell 的大小，以顯示更多資訊和程式碼。 您也可以使用最小化和最大化按鈕，在 Cloud Shell 和主要入口網站介面之間切換。

1. 執行下列指令，以建立包含專案的目錄，並變更為專案目錄。

    ```bash
    mkdir eventgrid
    cd eventgrid
    ```

1. 建立 .NET 主控台應用程式。

    ```bash
    dotnet new console
    ```

1. 執行下列命令，將 **Azure.Messaging.EventGrid** 和 **dotenv.net** 套件新增至專案。

    ```bash
    dotnet add package Azure.Messaging.EventGrid
    dotnet add package dotenv.net
    ```

### 設定主控台應用程式

在本節中，您會擷取主題端點和存取金鑰，以將其新增至 **.env** 檔案，進而保存這些秘密。

1. 執行下列命令，針對您先前建立的主題擷取 URL 和存取金鑰。 請務必記錄這些值。

    ```bash
    az eventgrid topic show --name $topicName -g $resourceGroup --query "endpoint" --output tsv
    az eventgrid topic key list --name $topicName -g $resourceGroup --query "key1" --output tsv
    ```

1. 執行下列命令建立 **.env** 檔案以保存秘密，然後在程式碼編輯器中開啟檔案。

    ```bash
    touch .env
    code .env
    ```

1. 將下列程式碼新增至 **.env** 檔案。 將 **YOUR_TOPIC_ENDPOINT** 和 **YOUR_TOPIC_ACCESS_KEY** 取代為您先前記錄的值。

    ```
    TOPIC_ENDPOINT="YOUR_TOPIC_ENDPOINT"
    TOPIC_ACCESS_KEY="YOUR_TOPIC_ACCESS_KEY"
    ```

1. 按 **ctrl+s** 以儲存檔案，然後按 **ctrl+q** 以結束編輯器。

現在您可以使用 Cloud Shell 中的編輯器取代 **Program.cs** 檔案中的範本程式碼。

### 新增專案的程式碼

1. 在 Cloud Shell 中執行下列命令，開始編輯應用程式。

    ```bash
    code Program.cs
    ```

1. 將任何現有程式碼取代為下列程式碼。 請務必檢閱程式碼中的註解。

    ```csharp
    using dotenv.net; 
    using Azure.Messaging.EventGrid; 
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    
    // Start the asynchronous process to send an Event Grid event
    ProcessAsync().GetAwaiter().GetResult();
    
    async Task ProcessAsync()
    {
        // Retrieve Event Grid topic endpoint and access key from environment variables
        var topicEndpoint = envVars["TOPIC_ENDPOINT"];
        var topicKey = envVars["TOPIC_ACCESS_KEY"];
        
        // Check if the required environment variables are set
        if (string.IsNullOrEmpty(topicEndpoint) || string.IsNullOrEmpty(topicKey))
        {
            Console.WriteLine("Please set TOPIC_ENDPOINT and TOPIC_ACCESS_KEY in your .env file.");
            return;
        }
    
        // Create an EventGridPublisherClient to send events to the specified topic
        EventGridPublisherClient client = new EventGridPublisherClient
            (new Uri(topicEndpoint),
            new Azure.AzureKeyCredential(topicKey));
    
        // Create a new EventGridEvent with sample data
        var eventGridEvent = new EventGridEvent(
            subject: "ExampleSubject",
            eventType: "ExampleEventType",
            dataVersion: "1.0",
            data: new { Message = "Hello, Event Grid!" }
        );
    
        // Send the event to Azure Event Grid
        await client.SendEventAsync(eventGridEvent);
        Console.WriteLine("Event sent successfully.");
    }
    ```

1. 按 **ctrl+s** 以儲存檔案，然後按 **ctrl+q** 以結束編輯器。

## 登入 Azure，然後執行應用程式

1. 在 Cloud Shell 命令行窗格中，輸入下列命令以登錄 Azure。

    ```
    az login
    ```

    **<font color="red">即使 Cloud Shell 工作階段已經過驗證，您還是必須登錄 Azure。</font>**

    > **注意**：在大部分情況下，只要使用 *az 登入*即可。 不過，如果您在多個租用戶中擁有訂用帳戶，您可能需要使用 *--tenant* 參數指定租用戶。 如需詳細資料，請參閱[使用 Azure CLI 以互動方式登入 Azure](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)。

1. 在 Cloud Shell 中執行下列命令以啟動主控台應用程式。 您會看見**已成功傳送事件**訊息。 傳送訊息後。

    ```bash
    dotnet run
    ```

1. 檢視您的 Web 應用程式，以查看您剛剛傳送的事件。 選取眼睛圖示以展開事件資料。

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。
1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。