---
lab:
  topic: Azure container services
  title: 使用 Azure Container Registry 工作建置和執行容器映像
  description: 了解如何使用 Azure CLI 命令來透過 Azure Container Registry 工作建置及執行容器映像。
---

# 使用 Azure Container Registry 工作建置和執行容器映像

在本練習中，您將從應用程式程式碼中建置容器映像，然後使用 Azure CLI 推送至 Azure Container Registry。 您將了解如何準備適用於容器化的應用程式、建立 ACR 執行個體，以及將容器映像儲存在 Azure。

在此練習中執行的工作：

* 建立 Azure Container Registry 資源
* 從 Dockerfile 建置和推送映像
* 驗證結果
* 在 Azure Container Registry 中執行映像

本練習大約需要**20** 分鐘才能完成。

## 建立 Azure Container Registry 資源

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站[https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。

1. 使用頁面上方搜尋欄右側的 **[\>_]** 按鈕，就能從 Azure 入口網站建立新的 Cloud Shell，並選取***Bash*** 環境。 Cloud Shell 會在 Azure 入口網站底部的窗格顯示命令列介面。 如果系統提示您選取儲存體帳戶以保存檔案，請選取 [不需要儲存體帳戶]****、[您的訂用帳戶]，然後選取 [套用]****。

    > **備註**：如果您之前就已建立使用*Bash* 環境的 Cloud Shell，請將原先的設定切換成***PowerShell***。

1. 針對本練習所需的資源建立資源群組。 將**myResourceGroup** 取代為您想要用於資源群組的名稱。 如有必要，您可以使用附近的地區來取代**eastus**。 如果您已有想要使用的資源群組，請繼續進行下一個步驟。

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. 執行下列命令以建立基本容器登錄。 登錄名稱在 Azure 內必須是唯一的，且包含 5-50 個數字與小寫字元。 將**myResourceGroup** 取代為您先前使用的名稱，並將**myContainerRegistry** 取代為唯一值。

    ```bash
    az acr create --resource-group myResourceGroup \
        --name myContainerRegistry --sku Basic
    ```

    > **注意：** 該命令會建立「基本」** 登錄，這是正在學習 Azure Container Registry 的開發人員適用的成本最佳化選項。

## 從 Dockerfile 建置和推送映像

接下來，您將根據 Dockerfile 建置和推送映像。

1. 執行下列命令以建立 Dockerfile。 Dockerfile 會包含單行，參考裝載於 Microsoft Container Registry 的*hello-world* 映像。

    ```bash
    echo FROM mcr.microsoft.com/hello-world > Dockerfile
    ```

1. 執行下列**az acr build** 命令來建置映射，並在成功建置映像後，將其推送至您的登錄。 以您先前建立的名稱取代**myContainerRegistry**。

    ```bash
    az acr build --image sample/hello-world:v1  \
        --registry myContainerRegistry \
        --file Dockerfile .
    ```

    以下是上一個命令輸出的簡短範例，其中顯示最後幾行與最終結果。 您可以在 [存放庫]** 欄位中看到*sample/hello-word* 映像已列出。

    ```
    - image:
        registry: myContainerRegistry.azurecr.io
        repository: sample/hello-world
        tag: v1
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      runtime-dependency:
        registry: mcr.microsoft.com
        repository: hello-world
        tag: latest
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      git: {}
    
    
    Run ID: cf1 was successful after 11s
    ```

## 驗證結果

1. 執行下列命令，列出登錄中的存放庫。 以您先前建立的名稱取代**myContainerRegistry**。

    ```bash
    az acr repository list --name myContainerRegistry --output table
    ```

    輸出：

    ```
    Result
    ----------------
    sample/hello-world
    ```

1. 執行下列命令，列出**sample/hello-world** 存放庫上的標籤。 以您先前使用的名稱取代**myContainerRegistry**。

    ```bash
    az acr repository show-tags --name myContainerRegistry \
        --repository sample/hello-world --output table
    ```

    輸出：

    ```
    Result
    --------
    v1
    ```

## 在 ACR 中執行映像

1. 使用**az acr run** 命令，從容器登錄執行*sample/hello-world:v1* 容器映像。 下列範例會使用 **$Registry**，指定要在其中執行命令的登錄。 以您先前使用的名稱取代**myContainerRegistry**。

    ```bash
    az acr run --registry myContainerRegistry \
        --cmd '$Registry/sample/hello-world:v1' /dev/null
    ```

    在此範例中的**cmd** 參數會執行其預設設定中的容器，但**cmd** 支援其他**docker run** 參數，甚至是其他**docker** 命令。 

    以下為縮點後的輸出範例：

    ```
    Packing source code into tar to upload...
    Uploading archived source code from '/tmp/run_archive_ebf74da7fcb04683867b129e2ccad5e1.tar.gz'...
    Sending context (1.855 KiB) to registry: mycontainerre...
    Queued a run with ID: cab
    Waiting for an agent...
    2019/03/19 19:01:53 Using acb_vol_60e9a538-b466-475f-9565-80c5b93eaa15 as the home volume
    2019/03/19 19:01:53 Creating Docker network: acb_default_network, driver: 'bridge'
    2019/03/19 19:01:53 Successfully set up Docker network: acb_default_network
    2019/03/19 19:01:53 Setting up Docker configuration...
    2019/03/19 19:01:54 Successfully set up Docker configuration
    2019/03/19 19:01:54 Logging in to registry: mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Successfully logged into mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Executing step ID: acb_step_0. Working directory: '', Network: 'acb_default_network'
    2019/03/19 19:01:55 Launching container with name: acb_step_0
    
    Hello from Docker!
    This message shows that your installation appears to be working correctly.
    
    2019/03/19 19:01:56 Successfully executed container: acb_step_0
    2019/03/19 19:01:56 Step ID: acb_step_0 marked as successful (elapsed time in seconds: 0.843801)
    
    Run ID: cab was successful after 6s
    ```

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站[https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。
1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。
