---
lab:
  topic: Azure Storage
  title: 使用 .NET 用戶端程式庫建立 Blob 儲存體資源
  description: 了解如何使用 Azure Storage .NET 用戶端程式庫來建立容器、上傳或列出 Blob，以及刪除容器。
---

# 使用 .NET 用戶端程式庫建立 Blob 儲存體資源

在本練習中，您將建立 Azure 儲存體帳戶，並使用 Azure 儲存體 Blob 用戶端程式庫建置 .NET 主控台應用程式，進而建立容器、將檔案上傳至 Blob 儲存體、列出 Blob 和下載檔案。 您將了解如何使用 Azure 進行驗證、以程式設計方式執行 Blob 儲存體作業，以及在 Azure 入口網站中驗證結果。

在此練習中執行的工作：

* 準備 Azure 資源
* 建立主控台應用程式以建立和資料
* 執行應用程式並驗證結果
* 清除資源

本練習大約需要 **30** 分鐘才能完成。

## 建立 Azure 儲存體帳戶

在本練習的此章節中，您將使用 Azure CLI 在 Azure 中建立所需的資源。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。

1. 使用頁面上方搜尋欄右側的 **[\>_]** 按鈕，就能從 Azure 入口網站建立新的 Cloud Shell，並選取 ***Bash*** 環境。 Cloud Shell 會在 Azure 入口網站底部的窗格顯示命令列介面。 如果系統提示您選取儲存體帳戶以保存檔案，請選取 [不需要儲存體帳戶]****、[您的訂用帳戶]，然後選取 [套用]****。

    > **備註**：如果您之前就已建立使用 *Bash* 環境的 Cloud Shell，請將原先的設定切換成 ***PowerShell***。

1. 在 Cloud Shell 工具列中，在**設定**功能表中，選擇**轉到經典版本**（這是使用程式碼編輯器所必需的）。

1. 針對本練習所需的資源建立資源群組。 將 **myResourceGroup** 取代為您想要用於資源群組的名稱。 如有必要，您可以使用附近的地區來取代 **eastus2**。 如果您已有想要使用的資源群組，請繼續進行下一個步驟。

    ```
    az group create --location eastus2 --name myResourceGroup
    ```

1. 許多命令需要唯一名稱，並使用相同的參數。 建立一些變數將減少建立資源的命令所需的變更。 執行下列命令來建立所需的變數。 將 **myResourceGroup** 取代為您用於此練習的名稱。

    ```
    resourceGroup=myResourceGroup
    location=eastus
    accountName=storageacct$RANDOM
    ```

1. 執行下列命令來建立 Azure 儲存體帳戶，每個帳戶名稱都必須是獨一無二的。 第一個命令會建立具有唯一名稱的變數，以用於您的儲存體帳戶。 從 **echo** 命令的輸出中記錄您的帳戶名稱。 

    ```
    az storage account create --name $accountName \
        --resource-group $resourceGroup \
        --location $location \
        --sku Standard_LRS 
    
    echo $accountName
    ```

### 將角色指派給 Microsoft Entra 使用者名稱

若要允許您的應用程式建立資源和項目，請將您的 Microsoft Entra 使用者指派給**儲存體 Blob 資料擁有者**角色。 在 Cloud Shell 中執行下列步驟。

>**提示：** 拖曳頂端框線調整 Cloud Shell 的大小，以顯示更多資訊和程式碼。 您也可以使用最小化和最大化按鈕，在 Cloud Shell 和主要入口網站介面之間切換。

1. 執行下列命令，從您的帳戶中擷取 **userPrincipalName**。 這代表角色將指派給誰。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 執行下列命令以擷取儲存體帳戶的資源識別碼。 資源識別碼會將角色指派的範圍設定為特定命名空間。

    ```
    resourceID=$(az storage account show --name $accountName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. 執行下列命令以建立和指派**儲存體 Blob 資料擁有者**角色。 此角色提供您管理容器和項目的權限。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Blob Data Owner" \
        --scope $resourceID
    ```

## 建立 .NET 主控台應用程式以建立容器和項目

現在所需的資源已部署至 Azure，下一個步驟是設定主控台應用程式。 下列步驟是在 Cloud Shell 中執行。

1. 執行下列指令，以建立包含專案的目錄，並變更為專案目錄。

    ```
    mkdir azstor
    cd azstor
    ```

1. 建立 .NET 主控台應用程式。

    ```
    dotnet new console
    ```

1. 執行下列命令，在應用程式中新增必要套件。

    ```
    dotnet add package Azure.Storage.Blobs
    dotnet add package Azure.Identity
    ```

1. 執行下列命令，在專案中建立 **data** 資料夾。 

    ```
    mkdir data
    ```

現在您可以新增專案程式碼。

### 新增專案的起始程式碼

1. 在 Cloud Shell 中執行下列命令，開始編輯應用程式。

    ```
    code Program.cs
    ```

1. 將任何現有內容取代為下列程式碼。 請務必檢閱程式碼中的註解。

    ```csharp
    using Azure.Storage.Blobs;
    using Azure.Storage.Blobs.Models;
    using Azure.Identity;
    
    Console.WriteLine("Azure Blob Storage exercise\n");
    
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Run the examples asynchronously, wait for the results before proceeding
    await ProcessAsync();
    
    Console.WriteLine("\nPress enter to exit the sample application.");
    Console.ReadLine();
    
    async Task ProcessAsync()
    {
        // CREATE A BLOB STORAGE CLIENT
        
    
    
        // CREATE A CONTAINER
        
    
    
        // CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE
        
    
    
        // UPLOAD THE FILE TO BLOB STORAGE
        
    
    
        // LIST BLOBS IN THE CONTAINER
        
    
    
        // DOWNLOAD THE BLOB TO A LOCAL FILE
        
    
    }
    ```

1. 按 **ctrl+s** 儲存變更，然後繼續執行下一個步驟。


## 新增程式碼以完成專案

在接下來的練習中，您將在指定區域中新增程式碼，以建立完整應用程式。 

1. 找到 **// CREATE A BLOB STORAGE CLIENT** 註解，然後在註解正下方直接新增下列程式碼。 **BlobServiceClient** 可作為管理儲存體帳戶中容器和 Blob 的主要入口點。 用戶端會使用 *DefaultAzureCredential* 進行驗證。 請務必將 **YOUR_ACCOUNT_NAME** 取代為您先前記錄的名稱。

    ```csharp
    // Create a credential using DefaultAzureCredential with configured options
    string accountName = "YOUR_ACCOUNT_NAME"; // Replace with your storage account name
    
    // Use the DefaultAzureCredential with the options configured at the top of the program
    DefaultAzureCredential credential = new DefaultAzureCredential(options);
    
    // Create the BlobServiceClient using the endpoint and DefaultAzureCredential
    string blobServiceEndpoint = $"https://{accountName}.blob.core.windows.net";
    BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), credential);
    ```

1. 按 **ctrl+s** 儲存變更，然後繼續執行下一個步驟。

1. 找到 **// CREATE A CONTAINER** 註解，然後在註解正下方直接新增下列程式碼。 建立容器包括建立 **BlobServiceClient** 類別的執行個體，然後呼叫 **CreateBlobContainerAsync** 方法，以在儲存體帳戶中建立容器。 GUID 值附加至容器名稱後，以確保其唯一性。 若容器已存在，**CreateBlobContainerAsync** 方法則會失敗。

    ```csharp
    // Create a unique name for the container
    string containerName = "wtblob" + Guid.NewGuid().ToString();
    
    // Create the container and return a container client object
    Console.WriteLine("Creating container: " + containerName);
    BlobContainerClient containerClient = 
        await blobServiceClient.CreateBlobContainerAsync(containerName);
    
    // Check if the container was created successfully
    if (containerClient != null)
    {
        Console.WriteLine("Container created successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("Failed to create the container, exiting program.");
        return;
    }
    ```

1. 按 **ctrl+s** 儲存變更，然後繼續執行下一個步驟。

1. 尋找 **// CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE** 註解，然後在註解正下方直接新增下列程式碼。 這會在資料目錄中建立上傳至容器的檔案。

    ```csharp
    // Create a local file in the ./data/ directory for uploading and downloading
    Console.WriteLine("Creating a local file for upload to Blob storage...");
    string localPath = "./data/";
    string fileName = "wtfile" + Guid.NewGuid().ToString() + ".txt";
    string localFilePath = Path.Combine(localPath, fileName);
    
    // Write text to the file
    await File.WriteAllTextAsync(localFilePath, "Hello, World!");
    Console.WriteLine("Local file created, press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. 按 **ctrl+s** 儲存變更，然後繼續執行下一個步驟。

1. 找到 **// UPLOAD THE FILE TO BLOB STORAGE** 註解，然後在註解正下方直接新增下列程式碼。 程式碼會透過呼叫在上一節建立容器上的 **GetBlobClient** 方法，以取得 **BlobClient** 物件的參考。 然後，程式碼會使用 **UploadAsync** 方法上傳產生的本機檔案。 如果 Blob 不存在，這個方法會建立 Blob，並在它確實存在時覆寫它。

    ```csharp
    // Get a reference to the blob and upload the file
    BlobClient blobClient = containerClient.GetBlobClient(fileName);
    
    Console.WriteLine("Uploading to Blob storage as blob:\n\t {0}", blobClient.Uri);
    
    // Open the file and upload its data
    using (FileStream uploadFileStream = File.OpenRead(localFilePath))
    {
        await blobClient.UploadAsync(uploadFileStream);
        uploadFileStream.Close();
    }
    
    // Verify if the file was uploaded successfully
    bool blobExists = await blobClient.ExistsAsync();
    if (blobExists)
    {
        Console.WriteLine("File uploaded successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("File upload failed, exiting program..");
        return;
    }
    ```

1. 按 **ctrl+s** 儲存變更，然後繼續執行下一個步驟。

1. 找到 **// LIST BLOBS IN THE CONTAINER** 註解，然後在註解正下方直接新增下列程式碼。 透過 **GetBlobsAsync** 方法，列出容器中的 Blob。 在這種情況下，只有一個 blob 被新增至容器中，因此清單作業只傳回該 blob。 

    ```csharp
    Console.WriteLine("Listing blobs in container...");
    await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
    {
        Console.WriteLine("\t" + blobItem.Name);
    }
    
    Console.WriteLine("Press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. 按 **ctrl+s** 儲存變更，然後繼續執行下一個步驟。

1. 找到 **// DOWNLOAD THE BLOB TO A LOCAL FILE** 註解，然後在註解下方直接新增下列程式碼。 程式碼會使用 **DownloadAsync** 方法，將先前建立的 Blob 下載至您的本機檔案系統。 範例程式碼會將 「DOWNLOADED」 的尾碼新增至 Blob 名稱，讓您可以在本機檔案系統中看到這兩個檔案。 

    ```csharp
    // Adds the string "DOWNLOADED" before the .txt extension so it doesn't 
    // overwrite the original file
    
    string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");
    
    Console.WriteLine("Downloading blob to: {0}", downloadFilePath);
    
    // Download the blob's contents and save it to a file
    BlobDownloadInfo download = await blobClient.DownloadAsync();
    
    using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath))
    {
        await download.Content.CopyToAsync(downloadFileStream);
    }
    
    Console.WriteLine("Blob downloaded successfully to: {0}", downloadFilePath);
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

1. 展開左側導覽中的 [> 資料儲存]****，然後選取 [容器]****。

1. 選取應用程式建立的容器，您可以檢視已上傳的 Blob。

1. 執行下列兩個命令，進入 **data** 目錄，並列出已上傳和下載的檔案。

    ```
    cd data
    ls
    ```

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。
1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。

