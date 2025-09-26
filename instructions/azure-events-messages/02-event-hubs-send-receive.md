---
lab:
  topic: Azure events and messaging
  title: 從 Azure 事件中樞傳送和擷取事件
  description: 了解如何使用 .NET Azure.Messaging.EventHubs SDK 從 Azure 事件中樞傳送和擷取事件。
---

# 從 Azure 事件中樞傳送和擷取事件

在本練習中，您將建立和設定 Azure 事件中樞資源，然後建置 .NET 主控台應用程式，以使用 **Azure.Messaging.EventHubs** SDK 傳送和接收訊息。 您將了解如何佈建雲端資源、與事件中樞互動，以及在完成後清理您的環境。

在此練習中執行的工作：

* 建立資源群組
* 建立 Azure 事件中樞資源
* 建立 .NET 主控台應用程式以傳送和擷取事件
* 清除資源

本練習大約需要 **30** 分鐘才能完成。

## 建立 Azure 事件中樞資源

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
    namespaceName=eventhubsns$RANDOM
    ```

### 建立 Azure 事件中樞命名空間和事件中樞

Azure 事件中樞命名空間是 Azure 內事件中樞資源的邏輯容器。 該命名空間提供唯一範圍容器，您可以在其中建立一或多個事件中樞，用來內嵌、處理及儲存大量事件資料。 下列指示會在 Cloud Shell 中執行。 

1. 執行下列命令以建立事件中樞命名空間。

    ```
    az eventhubs namespace create --name $namespaceName --resource-group $resourceGroup -l $location
    ```

1. 執行下列命令，在事件中樞命名空間中建立名為 **myEventHub** 的事件中樞。 

    ```
    az eventhubs eventhub create --name myEventHub --resource-group $resourceGroup \
      --namespace-name $namespaceName
    ```

### 將角色指派給 Microsoft Entra 使用者名稱

若要允許應用程式傳送和接收訊息，請將您的 Microsoft Entra 使用者指派給事件中樞命名空間層級的 **Azure 事件中樞資料擁有者**角色。 這將提供您的使用者帳戶使用 Azure RBAC 管理及存取佇列和主題的權限。 在 Cloud Shell 中執行下列步驟。

1. 執行下列命令，從您的帳戶中擷取 **userPrincipalName**。 這代表角色將指派給誰。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 執行下列命令以擷取事件中樞命名空間的資源識別碼。 資源識別碼會將角色指派的範圍設定為特定命名空間。

    ```
    resourceID=$(az eventhubs namespace show --resource-group $resourceGroup \
        --name $namespaceName --query id --output tsv)
    ```
1. 執行下列命令來建立和指派 **Azure 事件中樞資料擁有者**角色，這將授與您傳送和擷取事件的權限。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Event Hubs Data Owner" \
        --scope $resourceID
    ```

## 使用 .NET 主控台應用程式傳送和擷取事件

現在所需的資源已部署至 Azure，下一個步驟是設定主控台應用程式。 下列步驟是在 Cloud Shell 中執行。

>**提示：** 拖曳頂端框線調整 Cloud Shell 的大小，以顯示更多資訊和程式碼。 您也可以使用最小化和最大化按鈕，在 Cloud Shell 和主要入口網站介面之間切換。

1. 執行下列指令，以建立包含專案的目錄，並變更為專案目錄。

    ```
    mkdir eventhubs
    cd eventhubs
    ```

1. 建立 .NET 主控台應用程式。

    ```
    dotnet new console
    ```

1. 執行下列命令，將 **Azure.Messaging.EventHubs** 和 **Azure.Identity** 套件新增至專案。

    ```
    dotnet add package Azure.Messaging.EventHubs
    dotnet add package Azure.Identity
    ```

現在您可以使用 Cloud Shell 中的編輯器取代 **Program.cs** 檔案中的範本程式碼。

### 新增專案的起始程式碼

1. 在 Cloud Shell 中執行下列命令，開始編輯應用程式。

    ```
    code Program.cs
    ```

1. 將任何現有內容取代為下列程式碼。 請務必檢閱程式碼中的註解，並將 **YOUR_EVENT_HUB_NAMESPACE** 取代為您的事件中樞命名空間。

    ```csharp
    using Azure.Messaging.EventHubs;
    using Azure.Messaging.EventHubs.Producer;
    using Azure.Messaging.EventHubs.Consumer;
    using Azure.Identity;
    using System.Text;
    
    // TO-DO: Replace YOUR_EVENT_HUB_NAMESPACE with your actual Event Hub namespace
    string namespaceURL = "YOUR_EVENT_HUB_NAMESPACE.servicebus.windows.net";
    string eventHubName = "myEventHub"; 
    
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Number of events to be sent to the event hub
    int numOfEvents = 3;
    
    // CREATE A PRODUCER CLIENT AND SEND EVENTS
    
    
    
    // CREATE A CONSUMER CLIENT AND RECEIVE EVENTS
    
    
    ```

1. 按 **ctrl+s** 以儲存變更。

### 新增程式碼以完成應用程式

在本節中，您將新增程式碼以建立生產者和取用者用戶端，進而傳送和接收事件。

1. 找到 **// CREATE A PRODUCER CLIENT AND SEND EVENTS** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼中的註解。

    ```csharp
    // Create a producer client to send events to the event hub
    EventHubProducerClient producerClient = new EventHubProducerClient(
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    // Create a batch of events 
    using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();
    
    
    // Adding a random number to the event body and sending the events. 
    var random = new Random();
    for (int i = 1; i <= numOfEvents; i++)
    {
        int randomNumber = random.Next(1, 101); // 1 to 100 inclusive
        string eventBody = $"Event {randomNumber}";
        if (!eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes(eventBody))))
        {
            // if it is too large for the batch
            throw new Exception($"Event {i} is too large for the batch and cannot be sent.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of events to the event hub
        await producerClient.SendAsync(eventBatch);
    
        Console.WriteLine($"A batch of {numOfEvents} events has been published.");
        Console.WriteLine("Press Enter to retrieve and print the events...");
        Console.ReadLine();
    }
    finally
    {
        await producerClient.DisposeAsync();
    }
    ```

1. 按 **ctrl+s** 以儲存變更。

1. 找到 **// CREATE A CONSUMER CLIENT AND RETRIEVE EVENTS** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼中的註解。

    ```csharp
    // Create an EventHubConsumerClient
    await using var consumerClient = new EventHubConsumerClient(
        EventHubConsumerClient.DefaultConsumerGroupName,
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    Console.Clear();
    Console.WriteLine("Retrieving all events from the hub...");
    
    // Get total number of events in the hub by summing (last - first + 1) for all partitions
    // This count is used to determine when to stop reading events
    long totalEventCount = 0;
    string[] partitionIds = await consumerClient.GetPartitionIdsAsync();
    foreach (var partitionId in partitionIds)
    {
        PartitionProperties properties = await consumerClient.GetPartitionPropertiesAsync(partitionId);
        if (!properties.IsEmpty && properties.LastEnqueuedSequenceNumber >= properties.BeginningSequenceNumber)
        {
            totalEventCount += (properties.LastEnqueuedSequenceNumber - properties.BeginningSequenceNumber + 1);
        }
    }
    
    // Start retrieving events from the event hub and print to the console
    int retrievedCount = 0;
    await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsAsync(startReadingAtEarliestEvent: true))
    {
        if (partitionEvent.Data != null)
        {
            string body = Encoding.UTF8.GetString(partitionEvent.Data.Body.ToArray());
            Console.WriteLine($"Retrieved event: {body}");
            retrievedCount++;
            if (retrievedCount >= totalEventCount)
            {
                Console.WriteLine("Done retrieving events. Press Enter to exit...");
                Console.ReadLine();
                return;
            }
        }
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

1. 執行下列命令以啟動應用程式：

    ```
    dotnet run
    ```

    幾秒鐘後，您應該會看到類似下列範例的輸出：
    
    ```
    A batch of 3 events has been published.
    Press Enter to retrieve and print the events...
    
    Retrieving all events from the hub...
    Retrieved event: Event 4
    Retrieved event: Event 96
    Retrieved event: Event 74
    Done retrieving events. Press Enter to exit...
    ```

應用程式一律會將三個事件傳送至中樞，但會擷取中樞中的所有事件。 若您多次執行應用程式，則會擷取越來越多的事件。 用於建立事件的隨機數字可協助您識別不同事件。

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。
1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。 
