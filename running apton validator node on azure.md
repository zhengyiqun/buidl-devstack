# Azure

这是一个在Azure上安装Aptos节点的分步指南。按照这些步骤，在不同的机器上配置一个验证器节点和一个验证器全节点。

> **Did you set up your Azure account?**
> 
> This guide assumes that you already have Azure account setup.

## 在您继续开始之前

请确保你在继续之前完成这些先决条件的步骤。

- **Aptos CLI**: https://aptos.dev/cli-tools/aptos-cli-tool/install-aptos-cli
- **Terraform 1.2.4**: https://www.terraform.io/downloads.html
- **Kubernetes CLI**: https://kubernetes.io/docs/tasks/tools/
- **Azure CLI**: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli


## 安装

> 一个验证器节点+一个验证器全节点
> 
> 按照以下说明，即**首先**在一台机器上运行验证器节点，**其次**在另一台机器上运行验证器全节点

</br>

1. 为你的配置创建一个工作目录
   - 选择一个工作区名称，例如，主网为“mainnet”，测试网为“testnet”，等等。 **注意**：这定义了 Terraform 工作空间名称，该名称又用于形成资源名称。
   </br>

    ```bash
    export WORKSPACE=mainnet
    ```
    </br>
    
   - 为工作区创建一个目录
    </br>
  
    ```bash
    mkdir -p ~/$WORKSPACE
    ```
    </br>

   - 为您的节点选择一个用户名，例如“alice”
    </br>

    ```bash
    export USERNAME=alice
    ```
    </br>

2. 创建一个blob存储容器，用于在Azure上存储Terraform状态，你可以在Azure用户界面上或通过命令来完成。
    </br>
   
    ```bash
    az group create -l <azure region> -n aptos-$WORKSPACE
    az storage account create -n <storage account name> -g aptos-$WORKSPACE -l <azure region> --sku Standard_LRS
    az storage container create -n <container name> --account-name <storage account name> --resource-group aptos-$WORKSPACE
    ```
    </br>
