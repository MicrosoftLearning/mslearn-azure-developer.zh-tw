---
lab:
  topic: Secure solutions in Azure
  title: 從 Azure Key Vault 建立及擷取祕密
  description: 了解如何使用 Azure CLI 並以程式設計方式建立金鑰保存庫、建立和擷取秘密。
---

# 從 Azure Key Vault 建立及擷取祕密

在本練習中，您將建立 Azure Key Vault、使用 Azure CLI 儲存秘密，以及建置可從金鑰保存庫建立和擷取秘密的 .NET 主控台應用程式。 您將了解如何設定驗證、以程式設計方式管理秘密，以及在完成後清理資源。  

在此練習中執行的工作：

* 建立 Azure Key Vault 資源
* 使用 Azure CLI 在金鑰保存庫中儲存祕密
* 建立 .NET 主控台應用程式以建立和檢索秘密
* 清除資源

本練習大約需要 **30** 分鐘才能完成。

## 建立 Azure Key Vault 資源並新增秘密

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
    keyVaultName=mykeyvaultname$RANDOM
    ```

1. 執行下列命令以取得金鑰保存庫的名稱，並記錄名稱。 您將在稍後於本練習中使用到該名稱。

    ```
    echo $keyVaultName
    ```

1. 執行下列命令以建立 Azure Key Vault 資源。 執行可能需要數分鐘的時間。

    ```
    az keyvault create --name $keyVaultName \
        --resource-group $resourceGroup --location $location
    ```

### 將角色指派給 Microsoft Entra 使用者名稱

若要建立和擷取秘密，請將您的 Microsoft Entra 使用者指派給**金鑰保存庫秘密人員**角色。 這將提供您的使用者帳戶設定、刪除和列出秘密的權限。 在一般案例中，您可能想要將**金鑰保存庫秘密人員**指派給一個群組，並將可取得和列出秘密的**金鑰保存庫秘密使用者**指派給另一個群組，以分隔建立/讀取動作。

1. 執行下列命令，從您的帳戶中擷取 **userPrincipalName**。 這代表角色將指派給誰。

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. 執行下列命令，以擷取金鑰保存庫的資源識別碼。 資源識別碼會將角色指派的範圍設定為特定金鑰保存庫。

    ```
    resourceID=$(az keyvault show --resource-group $resourceGroup \
        --name $keyVaultName --query id --output tsv)
    ```

1. 執行下列命令以建立並指派**金鑰保存庫秘密人員**角色。

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Key Vault Secrets Officer" \
        --scope $resourceID
    ```

接下來，將秘密新增至您建立的金鑰保存庫。

### 使用 Azure CLI 新增和擷取秘密

1. 執行下列命令來建立秘密。 

    ```
    az keyvault secret set --vault-name $keyVaultName \
        --name "MySecret" --value "My secret value"
    ```

1. 執行下列命令以擷取秘密，並驗證秘密是否已設定。

    ```
    az keyvault secret show --name "MySecret" --vault-name $keyVaultName
    ```

    此命令會傳回一些 JSON。 最後一行包含純文字密碼。 

    ```json
    "value": "My secret value"
    ```

## 建立 .NET 主控台應用程式以儲存和擷取秘密

現在所需的資源已部署至 Azure，下一個步驟是設定主控台應用程式。 下列步驟是在 Cloud Shell 中執行。

>**提示：** 拖曳頂端框線調整 Cloud Shell 的大小，以顯示更多資訊和程式碼。 您也可以使用最小化和最大化按鈕，在 Cloud Shell 和主要入口網站介面之間切換。

1. 執行下列指令，以建立包含專案的目錄，並變更為專案目錄。

    ```
    mkdir keyvault
    cd keyvault
    ```

1. 建立 .NET 主控台應用程式。

    ```
    dotnet new console
    ```

1. 執行下列命令，將 **Azure.Identity** 和 **Azure.Security.KeyVault.Secrets** 套件新增至專案。

    ```
    dotnet add package Azure.Identity
    dotnet add package Azure.Security.KeyVault.Secrets
    ```

### 新增專案的起始程式碼

1. 在 Cloud Shell 中執行下列命令，開始編輯應用程式。

    ```
    code Program.cs
    ```

1. 將任何現有內容取代為下列程式碼。 請務必將 **YOUR-KEYVAULT-NAME** 取代為實際的金鑰保存庫名稱。

    ```csharp
    using Azure.Identity;
    using Azure.Security.KeyVault.Secrets;
    
    // Replace YOUR-KEYVAULT-NAME with your actual Key Vault name
    string KeyVaultUrl = "https://YOUR-KEYVAULT-NAME.vault.azure.net/";
    
    
    // ADD CODE TO CREATE A CLIENT
    
    
    
    // ADD CODE TO CREATE A MENU SYSTEM
    
    
    
    // ADD CODE TO CREATE A SECRET
    
    
    
    // ADD CODE TO LIST SECRETS
    
    
    ```

1. 按 **ctrl+s** 以儲存變更。

### 新增程式碼以完成應用程式

現在您可以新增程式碼以完成應用程式。

1. 找到 **// ADD CODE TO CREATE A CLIENT** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼和註解。

    ```csharp
    // Configure authentication options for connecting to Azure Key Vault
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create the Key Vault client using the URL and authentication credentials
    var client = new SecretClient(new Uri(KeyVaultUrl), new DefaultAzureCredential(options));
    ```

1. 找到 **// ADD CODE TO CREATE A MENU SYSTEM** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼和註解。

    ```csharp
    // Main application loop - continues until user types 'quit'
    while (true)
    {
        // Display menu options to the user
        Console.Clear();
        Console.WriteLine("\nPlease select an option:");
        Console.WriteLine("1. Create a new secret");
        Console.WriteLine("2. List all secrets");
        Console.WriteLine("Type 'quit' to exit");
        Console.Write("Enter your choice: ");
    
        // Read user input and convert to lowercase for easier comparison
        string? input = Console.ReadLine()?.Trim().ToLower();
        
        // Check if user wants to exit the application
        if (input == "quit")
        {
            Console.WriteLine("Goodbye!");
            break;
        }
    
        // Process the user's menu selection
        switch (input)
        {
            case "1":
                // Call the method to create a new secret
                await CreateSecretAsync(client);
                break;
            case "2":
                // Call the method to list all existing secrets
                await ListSecretsAsync(client);
                break;
            default:
                // Handle invalid input
                Console.WriteLine("Invalid option. Please enter 1, 2, or 'quit'.");
                break;
        }
    }
    ```

1. 找到 **// ADD CODE TO CREATE A SECRET** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼和註解。

    ```csharp
    async Task CreateSecretAsync(SecretClient client)
    {
        try
        {
            Console.Clear();
            Console.WriteLine("\nCreating a new secret...");
            
            // Get the secret name from user input
            Console.Write("Enter secret name: ");
            string? secretName = Console.ReadLine()?.Trim();
    
            // Validate that the secret name is not empty
            if (string.IsNullOrEmpty(secretName))
            {
                Console.WriteLine("Secret name cannot be empty.");
                return;
            }
            
            // Get the secret value from user input
            Console.Write("Enter secret value: ");
            string? secretValue = Console.ReadLine()?.Trim();
    
            // Validate that the secret value is not empty
            if (string.IsNullOrEmpty(secretValue))
            {
                Console.WriteLine("Secret value cannot be empty.");
                return;
            }
    
            // Create a new KeyVaultSecret object with the provided name and value
            var secret = new KeyVaultSecret(secretName, secretValue);
            
            // Store the secret in Azure Key Vault
            await client.SetSecretAsync(secret);
    
            Console.WriteLine($"Secret '{secretName}' created successfully!");
            Console.WriteLine("Press Enter to continue...");
            Console.ReadLine();
        }
        catch (Exception ex)
        {
            // Handle any errors that occur during secret creation
            Console.WriteLine($"Error creating secret: {ex.Message}");
        }
    }
    ```

1. 找到 **// ADD CODE TO LIST SECRETS** 註解，並在註解之後直接新增下列程式碼。 請務必檢閱程式碼和註解。

    ```csharp
    async Task ListSecretsAsync(SecretClient client)
    {
        try
        {
            Console.Clear();
            Console.WriteLine("Listing all secrets in the Key Vault...");
            Console.WriteLine("----------------------------------------");
    
            // Get an async enumerable of all secret properties in the Key Vault
            var secretProperties = client.GetPropertiesOfSecretsAsync();
            bool hasSecrets = false;
    
            // Iterate through each secret property to retrieve full secret details
            await foreach (var secretProperty in secretProperties)
            {
                hasSecrets = true;
                try
                {
                    // Retrieve the actual secret value and metadata using the secret name
                    var secret = await client.GetSecretAsync(secretProperty.Name);
                    
                    // Display the secret information to the console
                    Console.WriteLine($"Name: {secret.Value.Name}");
                    Console.WriteLine($"Value: {secret.Value.Value}");
                    Console.WriteLine($"Created: {secret.Value.Properties.CreatedOn}");
                    Console.WriteLine("----------------------------------------");
                }
                catch (Exception ex)
                {
                    // Handle errors for individual secrets (e.g., access denied, secret not found)
                    Console.WriteLine($"Error retrieving secret '{secretProperty.Name}': {ex.Message}");
                    Console.WriteLine("----------------------------------------");
                }
            }
    
            // Inform user if no secrets were found in the Key Vault
            if (!hasSecrets)
            {
                Console.WriteLine("No secrets found in the Key Vault.");
            }
        }
        catch (Exception ex)
        {
            // Handle general errors that occur during the listing operation
            Console.WriteLine($"Error listing secrets: {ex.Message}");
        
        }
        Console.WriteLine("Press Enter to continue...");
        Console.ReadLine();
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

1. 執行下列命令以啟動主控台應用程式。 應用程式會顯示應用程式的功能表系統。 

    ```
    dotnet run
    ```

1. 您已在本練習開始時建立秘密，請輸入 **2** 以擷取並顯示該秘密。

1. 輸入 **1**，並輸入秘密名稱和值以建立新秘密。

1. 再次列出秘密以檢視新增內容。

完成應用程式後，輸入 **quit**。

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。
1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。
