---
lab:
  topic: Azure container services
  title: 使用 Azure CLI 將容器部署至 Azure 容器應用程式
  description: 了解如何使用 Azure CLI 命令來建立安全的 Azure 容器應用程式環境，以及部署容器。
---

# 使用 Azure CLI 將容器部署至 Azure 容器應用程式

在本練習中，您將使用 Azure CLI 將容器化應用程式部署至 Azure 容器應用程式。 您將了解如何建立容器應用程式環境、部署容器，以及確認您的應用程式在 Azure 中執行。

在此練習中執行的工作：

* 在 Azure 中建立資源
* 建立容器應用程式環境
* 將容器應用程式部署至環境

本練習大約需要 **15** 分鐘才能完成。

## 建立資源群組並準備 Azure 環境

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。

1. 使用頁面上方搜尋欄右側的 **[\>_]** 按鈕，就能從 Azure 入口網站建立新的 Cloud Shell，並選取 ***Bash*** 環境。 Cloud Shell 會在 Azure 入口網站底部的窗格顯示命令列介面。 如果系統提示您選取儲存體帳戶以保存檔案，請選取 [不需要儲存體帳戶]****、[您的訂用帳戶]，然後選取 [套用]****。

    > **備註**：如果您之前就已建立使用 *Bash* 環境的 Cloud Shell，請將原先的設定切換成 ***PowerShell***。

1. 針對本練習所需的資源建立資源群組。 將 **myResourceGroup** 取代為您想要用於資源群組的名稱。 如有必要，您可以使用附近的地區來取代 **eastus**。 如果您已有想要使用的資源群組，請繼續進行下一個步驟。

    ```azurecli
    az group create --location eastus --name myResourceGroup
    ```

1. 執行下列命令，確定您已安裝 CLI 的最新版本 Azure 容器應用程式延伸模組。

    ```azurecli
    az extension add --name containerapp --upgrade
    ```

### 註冊命名空間

有兩個命名空間需要註冊 Azure 容器應用程式，並確定其已在下列步驟中註冊。 如果這些註冊尚未在您的訂月中註冊，則每個註冊可能需要幾分鐘才能完成。 

1. 註冊 **Microsoft.App** 命名空間。 

    ```bash
    az provider register --namespace Microsoft.App
    ```

1. 如果您之前尚未使用 Azure 監視器 Log Analytics 工作區，請註冊 **Microsoft.OperationalInsights** 提供者。

    ```bash
    az provider register --namespace Microsoft.OperationalInsights
    ```

## 建立容器應用程式環境

Azure 容器應用程式中的環境會在容器應用程式群組周圍建立安全界限。 部署至相同環境的容器應用程式會部署在相同的虛擬網路中，並將記錄寫入相同的 Log Analytics 工作區。

1. 使用 **az containerapp env create** 命令建立環境。 將 **myResourceGroup** 和 **myLocation** 取代為先前使用的值。 完成此作業需要幾分鐘。

    ```bash
    az containerapp env create \
        --name my-container-env \
        --resource-group myResourceGroup \
        --location myLocation
    ```

## 將容器應用程式部署至環境

容器應用程式環境完成部署之後，您可將容器映像部署至環境。

1. 使用 **containerapp create** 命令部署範例應用程式容器映像。 將 **myResourceGroup** 取代為先前使用的值。

    ```bash
    az containerapp create \
        --name my-container-app \
        --resource-group myResourceGroup \
        --environment my-container-env \
        --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
        --target-port 80 \
        --ingress 'external' \
        --query properties.configuration.ingress.fqdn
    ```

    將 **--ingress** 設定為 **external**，可讓容器應用程式接受公用要求。 此命令會傳回可存取應用程式的連結。

    ```
    Container app created. Access your app at <url>
    ```

若要驗證部署，請選取 **az containerapp create** 命令傳回的 URL，確認容器應用程式是否正在執行。

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。
1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。
