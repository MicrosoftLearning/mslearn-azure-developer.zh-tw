---
lab:
  topic: Azure events and messaging
  title: 從 Azure 佇列儲存體傳送和接收訊息
  description: 了解如何使用 .NET Azure.StorageQueues SDK 從 Azure 佇列儲存體傳送和接收訊息。
---

# 從 Azure 佇列儲存體傳送和接收訊息

在本練習中，您將建立和設定 Azure 佇列儲存體資源，然後建置 .NET 應用程式，以使用 **Azure.Storage.Queues** SDK 傳送和接收訊息。 您將了解如何佈建儲存資源、管理佇列訊息，以及在完成後清理環境。 

在此練習中執行的工作：

* 建立 Azure 佇列儲存體資源
* 將角色指派給 Microsoft Entra 使用者名稱
* 建立 .NET 主控台應用程式以傳送和接收訊息
* 清除資源

本練習大約需要 **30** 分鐘才能完成。

## 建立 Azure 佇列儲存體資源

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
    storAcctName=storactname$RANDOM
    ```

1. 稍後在本練習中，您將需要指派給儲存體帳戶的名稱。 執行下列命令並記錄輸出。

    ```
    echo $storAcctName
    ```

1. 執行下列命令，使用您先前建立的變數來建立儲存體帳戶。 此作業需要幾分鐘才能完成。

    ```bash
    az storage account create --resource-group $resourceGroup \
        --name $storAcctName --location $location --sku Standard_LRS
    ```

### 將角色指派給 Microsoft Entra 使用者名稱

若要允許您的應用程式傳送和接收訊息，請將您的 Microsoft Entra 使用者指派給**儲存體佇列資料參與者**角色。 這將提供您的使用者帳戶建立佇列的權限，以及使用 Azure RBAC 傳送/接收訊息。 在 Cloud Shell 中執行下列步驟。

1. 執行下列命令，從您的帳戶中擷取 **userPrincipalName**。 這代表角色將指派給誰。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 執行下列命令以擷取儲存體帳戶的資源識別碼。 資源識別碼會將角色指派的範圍設定為特定命名空間。

    ```
    resourceID=$(az storage account show --resource-group $resourceGroup \
        --name $storAcctName --query id --output tsv)
    ```

1. 執行下列命令以建立並指派**儲存體佇列資料參與者**角色。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Queue Data Contributor" \
        --scope $resourceID
    ```

## 建立 .NET 主控台應用程式以傳送和接收訊息

現在所需的資源已部署至 Azure，下一個步驟是設定主控台應用程式。 下列步驟是在 Cloud Shell 中執行。

>**提示：** 拖曳頂端框線調整 Cloud Shell 的大小，以顯示更多資訊和程式碼。 您也可以使用最小化和最大化按鈕，在 Cloud Shell 和主要入口網站介面之間切換。

1. 執行下列指令，以建立包含專案的目錄，並變更為專案目錄。

    ```
    mkdir queuestor
    cd queuestor
    ```

1. 建立 .NET 主控台應用程式。

    ```
    dotnet new console
    ```

1. 執行下列命令，將 **Azure.Storage.Queues** 和 **Azure.Identity** 套件新增至專案。

    ```
    dotnet add package Azure.Storage.Queues
    dotnet add package Azure.Identity
    ```

### 新增專案的起始程式碼

1. 在 Cloud Shell 中執行下列命令，開始編輯應用程式。

    ```
    code Program.cs
    ```

1. 將任何現有內容取代為下列程式碼。 請務必檢閱程式碼中的註解，並將 **<YOUR-STORAGE-ACCT-NAME>** 取代為先前記錄的儲存體帳戶名稱。

    ```csharp
    using Azure;
    using Azure.Identity;
    using Azure.Storage.Queues;
    using Azure.Storage.Queues.Models;
    using System;
    using System.Threading.Tasks;
    
    // Create a unique name for the queue
    // TODO: Replace the <YOUR-STORAGE-ACCT-NAME> placeholder 
    string queueName = "myqueue-" + Guid.NewGuid().ToString();
    string storageAccountName = "<YOUR-STORAGE-ACCT-NAME>";
    
    // ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE
    
    
    
    // ADD CODE TO SEND AND LIST MESSAGES
    
    
    
    // ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES
    
    
    
    // ADD CODE TO DELETE MESSAGES AND THE QUEUE
    
    
    ```

1. 按 **ctrl+s** 以儲存變更。

### 新增程式碼以建立佇列用戶端和佇列

現在您可以新增程式碼來建立佇列儲存體用戶端和建立佇列。

1. 找到 **// ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼和註解。

    ```csharp
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Instantiate a QueueClient to create and interact with the queue
    QueueClient queueClient = new QueueClient(
        new Uri($"https://{storageAccountName}.queue.core.windows.net/{queueName}"),
        new DefaultAzureCredential(options));
    
    Console.WriteLine($"Creating queue: {queueName}");
    
    // Create the queue
    await queueClient.CreateAsync();
    
    Console.WriteLine("Queue created, press Enter to add messages to the queue...");
    Console.ReadLine();
    ```

1. 按 **ctrl+s** 儲存檔案，然後繼續練習。

### 新增程式碼以傳送和列出佇列中的訊息

1. 找到 **// ADD CODE TO SEND AND LIST MESSAGES** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼和註解。

    ```csharp
    // Send several messages to the queue with the SendMessageAsync method.
    await queueClient.SendMessageAsync("Message 1");
    await queueClient.SendMessageAsync("Message 2");
    
    // Send a message and save the receipt for later use
    SendReceipt receipt = await queueClient.SendMessageAsync("Message 3");
    
    Console.WriteLine("Messages added to the queue. Press Enter to peek at the messages...");
    Console.ReadLine();
    
    // Peeking messages lets you view the messages without removing them from the queue.
    
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to update a message in the queue...");
    Console.ReadLine();
    ```

1. 按 **ctrl+s** 儲存檔案，然後繼續練習。

### 新增程式碼以更新訊息並列出結果

1. 找到 **// ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼和註解。

    ```csharp
    // Update a message with the UpdateMessageAsync method and the saved receipt
    await queueClient.UpdateMessageAsync(receipt.MessageId, receipt.PopReceipt, "Message 3 has been updated");
    
    Console.WriteLine("Message three updated. Press Enter to peek at the messages again...");
    Console.ReadLine();
    
    
    // Peek messages from the queue to compare updated content
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to delete messages from the queue...");
    Console.ReadLine();
    ```

1. 按 **ctrl+s** 儲存檔案，然後繼續練習。

### 新增程式碼以刪除訊息和佇列

1. 找到 **// ADD CODE TO DELETE MESSAGES AND THE QUEUE** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼和註解。

    ```csharp
    // Delete messages from the queue with the DeleteMessagesAsync method.
    foreach (var message in (await queueClient.ReceiveMessagesAsync(maxMessages: 10)).Value)
    {
        // "Process" the message
        Console.WriteLine($"Deleting message: {message.MessageText}");
    
        // Let the service know we're finished with the message and it can be safely deleted.
        await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    }
    Console.WriteLine("Messages deleted from the queue.");
    Console.WriteLine("\nPress Enter key to delete the queue...");
    Console.ReadLine();
    
    // Delete the queue with the DeleteAsync method.
    Console.WriteLine($"Deleting queue: {queueClient.Name}");
    await queueClient.DeleteAsync();
    
    Console.WriteLine("Done");
    ```

1. 按 **ctrl+s** 以儲存檔案，然後按 **ctrl+q** 以結束編輯器。

## 登入 Azure，然後執行應用程式

1. 在 Cloud Shell 命令行窗格中，輸入下列命令以登錄 Azure。

    ```
    az login
    ```

    **<font color="red">即使 Cloud Shell 工作階段已經過驗證，您還是必須登錄 Azure。</font>**

    > **注意**：在大部分情況下，只要使用 *az 登入*即可。 不過，如果您在多個租用戶中擁有訂用帳戶，您可能需要使用 *--tenant* 參數指定租用戶。 如需詳細資料，請參閱[使用 Azure CLI 以互動方式登入 Azure](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)。

1. 執行下列命令以啟動主控台應用程式。 該應用程式在執行過程中會暫停多次，並在您按下任意鍵後繼續執行。 這讓您有機會在 Azure 入口網站中檢視訊息。

    ```
    dotnet run
    ```

1. 在 Azure 入口網站中，瀏覽至您建立的 Azure 儲存體帳戶。 

1. 展開左側導覽中的 [> 資料儲存]****，然後選取 [佇列]****。

1. 選取應用程式建立的佇列，您可以檢視已傳送的訊息，並監視應用程式正在執行的動作。

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。
1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。

