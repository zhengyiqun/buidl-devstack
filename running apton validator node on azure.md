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


1. 为你的配置创建一个工作目录
    </br>
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

3. 在您的工作目录中创建名为 `main.tf` 的 Terraform 文件
    </br>

    ```bash
    cd ~/$WORKSPACE
    vi main.tf
    ```
    </br>

4. 修改 main.tf 文件以配置 Terraform，并从 Terraform 模块创建全节点。 `main.tf` 的示例内容：
    </br>
    ```
    terraform {
        required_version = "~> 1.2.0"
        backend "azurerm" {
        resource_group_name  = <resource group name>
        storage_account_name = <storage account name>
        container_name       = <container name>
        key                  = "state/validator"
        }
    }
    module "aptos-node" {
        # download Terraform module from aptos-labs/aptos-core repo
        source        = "github.com/aptos-labs/aptos-core.git//terraform/aptos-node/azure?ref=mainnet"
        region        = <azure region>  # Specify the region
        era           = 1              # bump era number to wipe the chain
        chain_id      = 43
        image_tag     = "mainnet" # Specify the docker image tag to use
        validator_name = "<Name of your validator>"
    }
    ```

    有关完整的自定义选项，请参阅变量文件 [here](https://github.com/aptos-labs/aptos-core/blob/main/terraform/aptos-node/azure/variables.tf), 和 [Helm values](https://github.com/aptos-labs/aptos-core/blob/main/terraform/helm/aptos-node/values.yaml).
    
    </br>

5. 在 `main.tf` 文件的同一目录中初始化 Terraform。
    </br>
    ```bash
    terraform init
    ```
    这将为您下载所有 Terraform 依赖项，位于当前工作目录的 `.terraform` 文件夹中。

    </br>

6. 创建一个新的 Terraform 工作区以隔离您的环境：
    </br>
    ```bash
    terraform workspace new $WORKSPACE
    # This command will list all workspaces
    terraform workspace list
    ```
    </br>

7. 应用配置
    </br>
    ```bash
    terraform apply
    ```
    这可能需要一段时间来完成（约20分钟），Terraform将在你的Azure账户中创建所有资源。

    </br>

8. 一旦terraform应用完成，你可以检查这些资源是否被创建
    </br>
    - `az aks get-credentials --resource-group aptos-$WORKSPACE --name aptos-$WORKSPACE` 为您的 k8s 集群配置访问权限
    - `kubectl get pods` 应该可以看到 haproxy、validator 和 fullnode。 validator 和 fullnode 的 pod 状态应该是 `pending` (需要在后面的步骤中进一步操作)
    - `kubectl get svc` 应该可以看到 `validator-lb` 和 `fullnode-lb`, 还有一个您可以稍后共享以进行连接的外部 IP 地址.
    </br>

9.  获取您的节点 IP 信息
    </br>
    ```bash
    export VALIDATOR_ADDRESS="$(kubectl get svc ${WORKSPACE}-aptos-node-0-validator-lb --output jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
    export FULLNODE_ADDRESS="$(kubectl get svc ${WORKSPACE}-aptos-node-0-fullnode-lb --output jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
    ```
    </br>

10. 在你的工作目录中生成密钥对（节点所有者、投票者、操作员密钥、共识密钥和网络密钥）。
    </br>

    ```bash
    aptos genesis generate-keys --output-dir ~/$WORKSPACE/keys
    ```
    </br>

    这将在 `~/$WORKSPACE/keys` 目录下创建 4 个密钥文件: 
      - `public-keys.yaml`
      - `private-keys.yaml`
      - `validator-identity.yaml`, and
      - `validator-full-node-identity.yaml`
  
    </br>

    > **注意**
    >
    > 将您的private-keys.yaml备份到安全的地方。这些密钥对于你建立节点的所有权非常重要。不要与任何人分享私人密钥.

    </br>

11. 配置验证者信息
    </br>
    
    ```bash
    aptos genesis set-validator-configuration \
      --local-repository-dir ~/$WORKSPACE \
      --username $USERNAME \
      --owner-public-identity-file ~/$WORKSPACE/keys/public-keys.yaml \
      --validator-host $VALIDATOR_ADDRESS:6180 \
      --full-node-host $FULLNODE_ADDRESS:6182 \
      --stake-amount 100000000000000
    ```
    </br>

    这将在 `~/$WORKSPACE/$USERNAME` 目录中创建两个 YAML 文件：`owner.yaml` 和 `operator.yaml`。

    </br>

12. 按照[Node Files](/nodes/node-files.md)页面上的下载命令，下载以下文件。
    - `genesis.blob`
    - `waypoint.txt`
    </br>

13. **总结：** 总结一下，在你的工作目录中，你应该有一个文件的列表。
    - `main.tf`: 安装`aptos-node`模块的Terraform文件（来自步骤3和4）
    - `keys` 文件夹包含:
      - `public-keys.yaml`: 所有者账户、共识、网络的公钥（来自步骤10）.
      - `private-keys.yaml`: 所有者账户、共识、网络的私钥（来自步骤10）.
      - `validator-identity.yaml`: 用于设置验证者身份的私钥（来自步骤10）.
      - `validator-full-node-identity.yaml`: 用于设置验证器全节点身份的私钥（来自步骤10）.
    - `username` 文件夹包含: 
      - `owner.yaml`: 定义了所有者、操作者和投票者的映射。它们都是测试模式下的相同账户（从第11步开始）.
      - `operator.yaml`: 将用于验证器和fullnode的节点信息（来自步骤11）。
    - `waypoint.txt`: 创世交易的航点（来自第12步）。
    - `genesis.blob`: 包含有关框架、validatorSet 等所有信息的 genesis 二进制文件（来自第 12 步）
    </br>

14. 将 "genesis.blob"、"waypoint.txt "和身份文件作为secret部署到k8s集群。
    </br>
    ```bash
    kubectl create secret generic ${WORKSPACE}-aptos-node-0-genesis-e1 \
        --from-file=genesis.blob=genesis.blob \
        --from-file=waypoint.txt=waypoint.txt \
        --from-file=validator-identity.yaml=keys/validator-identity.yaml \
        --from-file=validator-full-node-identity.yaml=keys/validator-full-node-identity.yaml
    ```
    </br>

    > **注意**
    >
    > `-e1`后缀指的是年代号。如果你改变了年代号，确保在创建秘密时与之相符

15. 检查所有 pod 是否都在运行
    </br>
    ```bash
    kubectl get pods
    NAME                                        READY   STATUS    RESTARTS   AGE
    node1-aptos-node-0-fullnode-e9-0              1/1     Running   0          4h31m
    node1-aptos-node-0-haproxy-7cc4c5f74c-l4l6n   1/1     Running   0          4h40m
    node1-aptos-node-0-validator-0                1/1     Running   0          4h30m
    ```
    </br>

现在你已经成功完成了节点的设置。请确保你已经设置了一台机器来运行验证器节点，以及第二台机器来运行验证器全节点。