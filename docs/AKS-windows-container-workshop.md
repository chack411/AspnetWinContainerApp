# Azure Kubernetes Service と Windows Container ワークショップ

Azure Kubernetes Service (AKS) は Azure でマネージドな Kubernetes クラスターを利用できるサービスです。 Window コンテナーをサポートしており、コンテナー化された .NET アプリを pod としてデプロイして、自動回復やスケールアウトなど、柔軟な運用が可能です。このワークショップでは、 Windows ノードを含む AKS クラスターを作成し、 ASP .NET　アプリをpodしてデプロイし、サービスの公開やスケーリング、およびAzure Monitorと連携したコンテナーの監視を実施します。

## 前提事項
本ワークショップは[ASP.NET と Windows Containers ワークショップ](container-tools.md)の実施を前提としており、このワークショップで作成したAzure Container Registry(ACR)やASP .NETコンテナーイメージを利用します。

## AKS クラスターの作成

AKS クラスターを作成し、Windows のノードプールを追加します

シェルを起動し、Azure CLIの`az aks create`を使用して AKS クラスターを作成します。 次の例では、`<myResourceGroup>` という名前のリソース グループに `<myAKSCluster>` という名前のクラスターを作成します。 このリソース グループは、[前のワークショップ](container-tools.md)でACR用に作成したリソースグループと同じものを利用します。次のコマンドではリージョンが指定されていませんので、AKS クラスターは指定したリソースグループのリージョンで作成されます。また、ACRからイメージをプルできるように、 AKSにACRをアタッチするオプション`--attach-acr`が付与されています。前のワークショップで作成したACRの名前を `<acrName>`に入力してください。

```azurecli
az aks create \
    --resource-group <myResourceGroup> \
    --name <myAKSCluster> \
    --node-count 1 \
    --generate-ssh-keys \
    --enable-addons monitoring \
#    --windows-admin-password $PASSWORD_WIN \
#    --windows-admin-username azureuser \
#    --vm-set-type VirtualMachineScaleSets \
#    --network-plugin azure
    --attach-acr <acrName>
```
作成したAKSクラスターでAzure Monitorを有効化するため、`--enable-addons`オプションを追加していますが、AKSクラスター作成後に有効化することも可能です。他にも多数のオプションを指定できます。`az aks create`コマンドの詳細は[こちら](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az_aks_create)を参照ください。

デプロイにはしばらく時間がかかります。デプロイが完了すると、この AKS デプロイに関する情報が JSON 形式で表示されます。

> 今回は実験用に1台のノードを指定していますが、が確実に動作するようにするには、少なくとも 3 つのノードの実行が望ましい構成です。

## Windowsノードプールの追加

AKSクラスターでは、同じ構成のノードを[ノードプール](https://docs.microsoft.com/ja-jp/azure/aks/use-multiple-node-pools)と呼ばる概念でグループ化して管理します。このため、構成の異なる複数種類のノード群で１つのAKSクラスターを構成することができます。AKSクラスターを作成すると、[システムノードプール](https://docs.microsoft.com/ja-jp/azure/aks/use-system-pools)と呼ばれるLinuxのノードが作成されます。このノードプールにはKubernetesに必須のコンポーネントが稼働するため、削除できません。
`az aks nodepool list`コマンドで、ノードプールの構成を確認できます。

```
az aks nodepool list \
    --resource-group <myResourceGroup> \
    --cluster-name <myAKSCluster> \
    -o table
```

今回はWindowsコンテナーを利用するため、Windowsのノードプールを追加します。ノードプールの追加には、`az aks nodepool add`コマンドを利用します。下記のコマンドでは、`npwin`という名前のWindows Serverノードプールを１台、AKSクラスターに追加します。

```
az aks nodepool add \
    --resource-group <myResourceGroup> \
    --cluster-name <myAKSCluster> \
    --os-type Windows \
    --name npwin \
    --node-count 1
```

ノードの追加にはしばらく時間がかかります。`az aks nodepool add`コマンドのオプションでVMのサイズなども指定できます。詳しくは[こちらのドキュメント](https://docs.microsoft.com/en-us/cli/azure/aks/nodepool?view=azure-cli-latest#az_aks_nodepool_add)を参照ください。

ノードの追加が完了したら、再び`az aks nodepool list`コマンドで状態を確認してみましょう。`npwin`という名前のWindowsノードプールが追加されていることが確認できます。

## kubectl を使用したクラスターへの接続

ここからは、Kubernetesの標準のコマンドラインツールである[kubectl](https://kubernetes.io/ja/docs/reference/kubectl/overview/)を用いて、Kubernetesを操作していきます。
Kubernetes クラスターに接続するように `kubectl` を構成するには、`az aks get-credentials`コマンドを使用します。 

```azurecli
az aks get-credentials --resource-group <myResourceGroup> --name <myAKSCluster>
```
このコマンドの実行により、クライアント環境にKubernetesに接続するための接続設定ファイルが作成されます。

クラスターへの接続を確認するには、クラスター ノードの一覧を返す `kubectl get nodes` コマンドを実行します。

```
$ kubectl get nodes -o wide
```

コマンドを実行すると、AKSクラスターのノードとして、LinuxのノードとWindowsのノードがそれぞれ１台ずつ稼働しているのが確認できます。

## AKSクラスターへのWindowsコンテナーのデプロイ

kubectlを利用して[前のワークショップ](container-tools.md)で作成したWindowsコンテナーイメージをAKSクラスターにデプロイします。
Kubernetesでのコンテナーのデプロイなど、様々なオブジェクトを作成するために、マニフェスト(yamlファイル）を作成します。下記はWindowsコンテナーをデプロイするDepolymentオブジェクトのマニフェストになります。<acrName>はWindowsコンテナーイメージが保存されているACRの名前に置き換えてください。


```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: winaspnetapp
  labels:
    app: winaspnetapp
spec:
  replicas: 2
  template:
    metadata:
      name: winaspnetapp
      labels:
        app: winaspnetapp
    spec:
      nodeSelector:
        "kubernetes.io/os": windows
      containers:
      - name: winaspnetapp
        image: <acrName>.azurecr.io/aspnetwincontainerapp:latest
        resources:
          limits:
            cpu: 1
            memory: 800M
          requests:
            cpu: .1
            memory: 300M
        ports:
          - containerPort: 80
  selector:
    matchLabels:
      app: winaspnetapp
EOF
```

このマニフェストにより、Windowsコンテナーイメージ`aspnetwincontainerapp:latest`を用いたpodが２つ、AKSクラスターにデプロイされます。注意すべき点は、このpodはWindowsノードでのみ稼働できる点です。AKSクラスターにはLinuxノードも稼働しているため、このpodをWindowsノードでのみスケジュールする必要があります。このために、マニフェストに`nodeSelector`として`"kubernetes.io/os": windows`のラベルが指定されています。これにより、podはこのラベルを持つwindowsノードにの実行されます。
下記のコマンドでAKSのノードに設定されたラベルが確認できます。

```
kubectl get node --show-labels
```

適切なノードにうまくpodがスケジュールされるようにする別の方法としては、`taint`を利用する方法があります。主にWindowsノードのみを利用する場合、Linuxノードに`taint`を設定することで、`toleration`が設定されていないpodはWindowsノードでのみ実行するようにできます。詳細は[こちら](https://kubernetes.io/ja/docs/concepts/scheduling-eviction/taint-and-toleration/)を参照ください。

## WindowsコンテナーアプリのAKSクラスター外部への公開

AKSにWindowsコンテナーのデプロイがきましたので、次にアプリを外部のクライアント環境からアクセスできるようにします。このためにKubernetesのserviceオブジェクトを次のマニフェストを用いて作成します。

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: winaspnetapp
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
  selector:
    app: winaspnetapp
EOF
```

次のコマンドで、作成されたserviceオブジェクトを確認します。

```
kubectl get service -w

```

上記コマンドの`-w`オプションで継続的に状態を確認できます。しばらくすると`EXTERNAL-IP`に値がパブリックIPアドレス表示されるので、ブラウザを開いて表示されたIPアドレスに接続してください。

画像

無事にWindowsコンテナーのアプリに接続できていることが確認できます。

## podのスケーリング




追加のチュートリアルでは、このアプリケーションがスケールアウトされて更新されます。

このクイックスタートは、Kubernetes の基本的な概念を理解していることを前提としています。 詳細については、「[Azure Kubernetes Services (AKS) における Kubernetes の中心概念][kubernetes-concepts]」を参照してください。

## <a name="before-you-begin"></a>開始する前に

前のチュートリアルでは、アプリケーションをコンテナー イメージにパッケージ化し、このイメージを Azure Container Registry にアップロードして、Kubernetes クラスターを作成しました。

このチュートリアルを完了するには、事前に作成した `azure-vote-all-in-one-redis.yaml` Kubernetes マニフェスト ファイルが必要です。 このファイルは、前のチュートリアルでは、アプリケーションのソース コードと共にダウンロードされました。 リポジトリの複製が作成されていること、およびディレクトリが複製されたディレクトリに変更されていることを確認します。 これらの手順を完了しておらず、順番に進めたい場合は、[チュートリアル 1 - コンテナー イメージを作成する][aks-tutorial-prepare-app]に関するページから開始してください。

このチュートリアルでは、Azure CLI バージョン 2.0.53 以降を実行している必要があります。 バージョンを確認するには、`az --version` を実行します。 インストールまたはアップグレードする必要がある場合は、[Azure CLI のインストール][azure-cli-install]に関するページを参照してください。

## <a name="update-the-manifest-file"></a>マニフェスト ファイルを更新する

これらのチュートリアルでは、Azure Container Registry (ACR) インスタンスがサンプル アプリケーション用のコンテナー イメージを格納しています。 アプリケーションをデプロイするには、Kubernetes マニフェスト ファイル内のイメージ名を、ACR ログイン サーバー名が含まれるように更新する必要があります。

次のように、[az acr list][az-acr-list] コマンドを使用して、ACR ログイン サーバー名を取得します。

```azurecli
az acr list --resource-group myResourceGroup --query "[].{acrLoginServer:loginServer}" --output table
```

最初のチュートリアルで複製した git repo のサンプル マニフェスト ファイルは、*microsoft* というログイン サーバー名を使用します。 現在の場所が、複製された *azure-voting-app-redis* ディレクトリ内であることを確認し、テキスト エディター (`vi` など) でマニフェスト ファイルを開きます。

```console
vi azure-vote-all-in-one-redis.yaml
```

*microsoft* は、実際の ACR ログイン サーバー名に置き換えてください。 イメージ名は、マニフェスト ファイルの 51 行目にあります。 次の例は、既定のイメージ名を示しています。

```yaml
containers:
- name: azure-vote-front
  image: microsoft/azure-vote-front:v1
```

マニフェスト ファイルが次の例のようになるように、独自の ACR ログイン サーバー名を指定します。

```yaml
containers:
- name: azure-vote-front
  image: <acrName>.azurecr.io/azure-vote-front:v1
```

ファイルを保存して閉じます。 `vi` では、`:wq` を使用します。

## <a name="deploy-the-application"></a>アプリケーションの配置

ご利用になるアプリケーションをデプロイするには、[kubectl apply][kubectl-apply] コマンドを使用します。 このコマンドは、マニフェスト ファイルを解析し、定義されている Kubernetes オブジェクトを作成します。 次の例に示すように、サンプルのマニフェスト ファイルを指定します。

```console
kubectl apply -f azure-vote-all-in-one-redis.yaml
```

次の出力例では、AKS クラスター内で正常に作成されたリソースが示されています。

```
$ kubectl apply -f azure-vote-all-in-one-redis.yaml

deployment "azure-vote-back" created
service "azure-vote-back" created
deployment "azure-vote-front" created
service "azure-vote-front" created
```

## <a name="test-the-application"></a>アプリケーションをテストする

アプリケーションが実行されると、Kubernetes サービスによってアプリケーション フロント エンドがインターネットに公開されます。 このプロセスが完了するまでに数分かかることがあります。

進行状況を監視するには、[kubectl get service][kubectl-get] コマンドを `--watch` 引数と一緒に使用します。

```console
kubectl get service azure-vote-front --watch
```

最初に、*azure-vote-front* サービスの *EXTERNAL-IP* が "*保留中*" として表示されます。

```
azure-vote-front   LoadBalancer   10.0.34.242   <pending>     80:30676/TCP   5s
```

*EXTERNAL-IP* アドレスが "*保留中*" から実際のパブリック IP アドレスに変わったら、`CTRL-C` を使用して `kubectl` ウォッチ プロセスを停止します。 次の出力例は、サービスに割り当てられている有効なパブリック IP アドレスを示しています。

```
azure-vote-front   LoadBalancer   10.0.34.242   52.179.23.131   80:30676/TCP   67s
```

アプリケーションが動作していることを確認するには、Web ブラウザーを開いてサービスの外部 IP アドレスにアクセスします。

![Azure 上の Kubernetes クラスターの図](media/container-service-kubernetes-tutorials/azure-vote.png)

アプリケーションが読み込まれなかった場合、イメージ レジストリに関する承認の問題が原因になっている可能性があります。 コンテナーのステータスを表示するには、`kubectl get pods` コマンドを使用します。 コンテナー イメージがプルできない場合は、「[Azure Kubernetes Service から Azure Container Registry の認証を受ける](cluster-container-registry-integration.md)」を参照してください。

## <a name="next-steps"></a>次のステップ

このチュートリアルでは、サンプルの Azure vote アプリケーションを AKS の Kubernetes クラスターにデプロイしました。 以下の方法を学習しました。

> [!div class="checklist"]
> * Kubernetes マニフェスト ファイルを更新する
> * Kubernetes でアプリケーションを実行する
> * アプリケーションをテストする

次のチュートリアルに進んで、Kubernetes アプリケーションとその基になっている Kubernetes インフラストラクチャのスケーリング方法に関して学習してください。

> [!div class="nextstepaction"]
> [Kubernetes ポッドと Kubernetes インフラストラクチャをスケーリングする][aks-tutorial-scale]

<!-- LINKS - external -->
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[kubectl-create]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create
[kubectl-get]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get

<!-- LINKS - internal -->
[aks-tutorial-prepare-app]: ./tutorial-kubernetes-prepare-app.md
[aks-tutorial-scale]: ./tutorial-kubernetes-scale.md
[az-acr-list]: /cli/azure/acr
[azure-cli-install]: /cli/azure/install-azure-cli
[kubernetes-concepts]: concepts-clusters-workloads.md
[kubernetes-service]: concepts-network.md#services
