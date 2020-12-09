﻿# Kubernetes on Azure のチュートリアル - クラスターのデプロイ

description: この Azure Kubernetes Service (AKS) のチュートリアルでは、AKS クラスターを作成し、kubectl を使用して Kubernetes マスター ノードに接続します。

# <a name="tutorial-deploy-an-azure-kubernetes-service-aks-cluster"></a>チュートリアル:Azure Kubernetes Service (AKS) クラスターのデプロイ

Kubernetes には、コンテナー化されたアプリケーション用の分散プラットフォームが用意されています。 AKS を使うと、運用開始準備の整った Kubernetes クラスターを迅速に作成できます。 このチュートリアル (7 部構成の第 3 部) では、Kubernetes クラスターを AKS にデプロイします。 学習内容は次のとおりです。

> [!div class="checklist"]
> * Azure Container Registry に対して認証できる Kubernetes AKS クラスターをデプロイする
> * Kubernetes CLI (kubectl) をインストールする
> * kubectl を構成して AKS クラスターに接続する

追加のチュートリアルでは、Azure Vote アプリケーションをクラスターにデプロイし、スケーリングおよび更新します。

## <a name="before-you-begin"></a>開始する前に

これまでのチュートリアルでは、コンテナー イメージを作成して、Azure Container Registry インスタンスにアップロードしました。 これらの手順を完了しておらず、順番に進めたい場合は、[チュートリアル 1 - コンテナー イメージを作成する][aks-tutorial-prepare-app]に関するページから開始してください。

このチュートリアルでは、Azure CLI バージョン 2.0.53 以降を実行している必要があります。 バージョンを確認するには、`az --version` を実行します。 インストールまたはアップグレードする必要がある場合は、[Azure CLI のインストール][azure-cli-install]に関するページを参照してください。

## <a name="create-a-kubernetes-cluster"></a>Kubernetes クラスターを作成する

AKS クラスターでは、Kubernetes のロールベースのアクセス制御 (RBAC) を使用できます。 これらのコントロールを使用すると、ユーザーに割り当てられているロールに基づいてリソースへのアクセスを定義できます。 ユーザーに複数のロールが割り当てられている場合は、アクセス許可が組み合わされます。また、アクセス許可のスコープを 1 つの名前空間またはクラスター全体に設定できます。 Azure CLI の既定では、AKS クラスターを作成するときに RBAC が自動的に有効になります。

[az aks create][] を使用して AKS クラスターを作成します。 次の例では、*myResourceGroup* という名前のリソース グループに *myAKSCluster* という名前のクラスターを作成します。 このリソース グループは、[前のチュートリアル][aks-tutorial-prepare-acr]で作成しました ("*米国東部*" リージョン)。 次の例ではリージョンが指定されず、AKS クラスターも "*米国東部*" リージョンで作成されます。 AKS のリソース制限とリージョン可用性に関する詳細については、「[Azure Kubernetes Service (AKS) のクォータ、仮想マシンのサイズの制限、利用可能なリージョン][quotas-skus-regions]」を参照してください。

AKS クラスターが他の Azure リソースと対話できるようにするために、Azure Active Directory のサービス プリンシパルが自動的に作成されます (指定しなかったため)。 この場合、このサービス プリンシパルは、前のチュートリアルで作成した Azure Container Registry (ACR) インスタンスから[イメージをプルする権利を付与][container-registry-integration]されます。 管理しやすくするために、サービス プリンシパルの代わりに[マネージド ID](use-managed-identity.md) を使用することもできることに注意してください。

```azurecli
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 2 \
    --generate-ssh-keys \
    --attach-acr <acrName>
```

ACR からイメージをプルするようにサービス プリンシパルを手動で構成することもできます。 詳細については、[サービス プリンシパルによる ACR 認証](../container-registry/container-registry-auth-service-principal.md)または[プル シークレットを使用した Kubernetes からの認証](../container-registry/container-registry-auth-kubernetes.md)に関するページを参照してください。

数分してデプロイが完了すると、この AKS デプロイに関する情報が JSON 形式で表示されます。

> [!NOTE]
> クラスターが確実に動作するようにするには、少なくとも 2 つのノードを実行する必要があります。

## <a name="install-the-kubernetes-cli"></a>Kubernetes CLI のインストール

お使いのローカル コンピューターから Kubernetes クラスターに接続するには、[kubectl][kubectl] (Kubernetes コマンドライン クライアント) を使用します。

Azure Cloud Shell を使用している場合、`kubectl` は既にインストールされています。 [az aks install-cli][] コマンドを使用してローカルにインストールすることもできます。

```azurecli
az aks install-cli
```

## <a name="connect-to-cluster-using-kubectl"></a>kubectl を使用したクラスターへの接続

Kubernetes クラスターに接続するように `kubectl` を構成するには、[az aks get-credentials][] コマンドを使用します。 次の例では、*myResourceGroup* の *myAKSCluster* という名前の AKS クラスターの資格情報を取得します。

```azurecli
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

クラスターへの接続を確認するには、クラスター ノードの一覧を返す [kubectl get nodes][kubectl-get] コマンドを実行します。

```
$ kubectl get nodes

NAME                       STATUS   ROLES   AGE   VERSION
aks-nodepool1-12345678-0   Ready    agent   32m   v1.14.8
```

## <a name="next-steps"></a>次のステップ

このチュートリアルでは、Kubernetes クラスターを AKS にデプロイし、`kubectl` を構成してクラスターに接続しました。 以下の方法を学習しました。

> [!div class="checklist"]
> * Azure Container Registry に対して認証できる Kubernetes AKS クラスターをデプロイする
> * Kubernetes CLI (kubectl) をインストールする
> * kubectl を構成して AKS クラスターに接続する

次のチュートリアルに進み、アプリケーションをクラスターにデプロイする方法を学習してください。

> [!div class="nextstepaction"]
> [Kubernetes でアプリケーションをデプロイする][aks-tutorial-deploy-app]

<!-- LINKS - external -->
[kubectl]: https://kubernetes.io/docs/user-guide/kubectl/
[kubectl-get]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get

<!-- LINKS - internal -->
[aks-tutorial-deploy-app]: ./tutorial-kubernetes-deploy-application.md
[aks-tutorial-prepare-acr]: ./tutorial-kubernetes-prepare-acr.md
[aks-tutorial-prepare-app]: ./tutorial-kubernetes-prepare-app.md
[az ad sp create-for-rbac]: /cli/azure/ad/sp#az-ad-sp-create-for-rbac
[az acr show]: /cli/azure/acr#az-acr-show
[az role assignment create]: /cli/azure/role/assignment#az-role-assignment-create
[az aks create]: /cli/azure/aks#az-aks-create
[az aks install-cli]: /cli/azure/aks#az-aks-install-cli
[az aks get-credentials]: /cli/azure/aks#az-aks-get-credentials
[azure-cli-install]: /cli/azure/install-azure-cli
[container-registry-integration]: ./cluster-container-registry-integration.md
[quotas-skus-regions]: quotas-skus-regions.md