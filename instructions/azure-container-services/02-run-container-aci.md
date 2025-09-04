---
lab:
  topic: Azure container services
  title: 使用 Azure CLI 命令將容器部署至 Azure 容器執行個體
  description: 了解如何使用 Azure CLI 命令將容器部署至 Azure 容器執行個體。
---

# 使用 Azure CLI 命令將容器部署至 Azure 容器執行個體

在本練習中，您可以使用 Azure CLI 在 Azure 容器執行個體 (ACI) 中部署及執行容器。 您將了解如何建立容器群組、指定容器設定，以及確認您的容器化應用程式正在雲端中執行。

在此練習中執行的工作：

* 在 Azure 中建立 Azure 容器執行個體資源
* 建立及部署容器
* 確認容器正在執行

本練習大約需要 **15** 分鐘才能完成。

## 建立資源群組

1. 在網頁瀏覽器中，瀏覽至 Azure 入口網站 [https://portal.azure.com](https://portal.azure.com)；若出現提示，請使用您的 Azure 認證登入。

1. 使用頁面上方搜尋欄右側的 **[\>_]** 按鈕，就能從 Azure 入口網站建立新的 Cloud Shell，並選取 ***Bash*** 環境。 Cloud Shell 會在 Azure 入口網站底部的窗格顯示命令列介面。 如果系統提示您選取儲存體帳戶以保存檔案，請選取 [不需要儲存體帳戶]****、[您的訂用帳戶]，然後選取 [套用]****。

    > **備註**：如果您之前就已建立使用 *Bash* 環境的 Cloud Shell，請將原先的設定切換成 ***PowerShell***。

1. 針對本練習所需的資源建立資源群組。 將 **myResourceGroup** 取代為您想要用於資源群組的名稱。 如有必要，您可以使用附近的地區來取代 **eastus**。 如果您已有想要使用的資源群組，請繼續進行下一個步驟。

    ```
    az group create --location eastus --name myResourceGroup
    ```

## 建立及部署容器

您可以向 **az container create** 命令提供名稱、Docker 映像與 Azure 資源群組來建立容器。 您會透過指定 DNS 名稱標籤，向網際網路公開容器。

1. 執行下列命令，以建立用來將容器公開給網際網路的 DNS 名稱。 您的 DNS 名稱必須是唯一的，請從 Cloud Shell 執行此命令，以建立保留唯一名稱的變數。

    ```bash
    DNS_NAME_LABEL=aci-example-$RANDOM
    ```

1. 執行下列命令以建立容器執行個體。 將 **myResourceGroup** 和 **myLocation** 取代為先前使用的值。 完成此作業需要幾分鐘。

    ```bash
    az container create --resource-group myResourceGroup \
        --name mycontainer \
        --image mcr.microsoft.com/azuredocs/aci-helloworld \
        --ports 80 \
        --dns-name-label $DNS_NAME_LABEL --location myLocation \
        --os-type Linux \
        --cpu 1 \
        --memory 1.5 
    ```

    在上一個命令中，**$DNS_NAME_LABEL** 會指定您的 DNS 名稱。 映像名稱 **mcr.microsoft.com/azuredocs/aci-helloworld** 指的是會執行基本 Node.js Web 應用程式的 Docker 映像。

在 **az container create** 命令完成之後，前往下一個區段。

## 確認容器正在執行

您可以使用 **az container show** 命令來檢查容器的建置狀態。 

1. 執行下列命令以檢查所建立容器的佈建狀態。 將 **myResourceGroup** 取代為先前使用的值。

    ```bash
    az container show --resource-group myResourceGroup \
        --name mycontainer \
        --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
        --out table 
    ```

    您會看到容器的完整網域名稱 (FQDN) 及其佈建狀態。 以下是範例。

    ```
    FQDN                                    ProvisioningState
    --------------------------------------  -------------------
    aci-wt.eastus.azurecontainer.io         Succeeded
    ```

    > **注意：** 如果容器處於**正在建立**狀態，請稍候片刻，然後再次執行命令，直到您看到**已成功**狀態。

1. 從瀏覽器中瀏覽至您容器的 FQDN 以查看其執行狀態。 您可能會收到網站不安全的警告。

## 清除資源

現在您已完成練習，您應該刪除建立的雲端資源，以避免不必要的資源使用狀況。

1. 瀏覽至您建立的資源群組，並檢視此練習中所使用的資源內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。

> **注意：** 刪除資源群組時，會刪除其中包含的所有資源。 如果您選擇此練習的現有資源群組，則本練習範圍外的任何現有資源也將遭到刪除。
