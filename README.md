# アプリ作成

1. ひな型の作成

    ```
    npx express-generator sample-webapp --view ejs
    cd sample-webapp
    npm install
    ```

1. ビルド、テストコマンドを `/package.json` に追加

    ```
    {
        ...省略...
        "scripts": {
            "start": "node ./bin/www",
            "build": "echo 'building ...'",
            "test": "echo 'testing ...'"
        },
        ...省略...
    }
    ```

# VM サーバー

1. 必要ミドルのインストール

    * Node.js

        ```
        curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
        yum install -y nodejs
        ```

1. アプリの転送と設定

    ```
    # フォルダ準備
    sudo mkdir /app
    sudo chmod +x /app

    # ファイル転送
    scp -i <PEMファイル> ./dist auzreuser@xxx.xxx.xxx.xxx:/app

    # サービス追加
    mv ./webapp.service /etc/systemd/system/
    cd /etc/systemd/system/
    chmod +x ./webapp.service
    chcon \
        -u system_u \
        -r object_r \
        -t systemd_unit_file_t \
        ./webapp.service
    
    systemctl daemon-reload
    systemctl enable webapp
    systemctl start webapp
    ```

# Self-hosted runner

1. 仮想マシン作成
1. 必要ミドルのインストール
    * Docker
        ```
        # Setup the repository
        sudo yum install -y yum-utils
        sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

        # Install Docker Engine
        sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
        sudo systemctl enable docker
        sudo systemctl start docker

        # Modify permissions
        sudo gpasswd -a $(whoami) docker
        sudo chgrp docker /var/run/docker.sock
        sudo service docker restart
        sudo ./svc.sh stop
        sudo ./svc.sh start
        ```

1. GitHub Actions Runner のインストール

    1. Runner本体のインストール

        ```
        mkdir actions-runner && cd actions-runner
        curl -o actions-runner-linux-x64-2.306.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.306.0/actions-runner-linux-x64-2.306.0.tar.gz
        tar xzf ./actions-runner-linux-x64-2.306.0.tar.gz
        ./config.sh --url https://github.com/YOUR_ACCOUNT/YOUR_REPO_NAME --token YOUR_TOKEN
        ```

    1. Runnderのサービス登録

        ```
        chcon system_u:object_r:usr_t:s0 runsvc.sh
        sudo ./svc.sh install
        ```

1. 実行

    ```
    sudo ./svc.sh start
    ```

# Azure AppServices

OpenID Connect を使ったデプロイ

## AzureAD にアプリを作成

アプリの作成

1. AzureADを開く
1. [管理]-[エンタープライズアプリケーション]を開く
1. 「新しいアプリケーション」を開く
1. 「独自のアプリケーションの作成」を選択
1. 以下を設定して「作成」
    * 名前： (任意。例： `Test Web Siete (AppService)`)
    * 操作： `アプリケーションを登録して Azure AD と統合(開発中のアプリ)`
1. 以下を設定して「登録」
    * 名前： (任意)
    * アカウント種類： `この組織ディレクトリに含まれるアカウント`

資格情報の発行

1. AzureADの `エンタープライズアプリケーション` から作成したアプリを開く
1. [管理]-[シングルサインオン]を開く
1. 「アプリケーションに移動」を押して移動
1. [管理]-[証明書とシークレット]を開く
1. 「フェデレーション資格情報」タブへ移動
1. 「資格情報の追加」
1. 以下を設定して「追加」
    * シナリオ： `AzureリソースをデプロイするGitHub Actions`
    * GitHubアカウント
        * 組織： (GitHub の 組織またはユーザー)
        * リポジトリ： (GitHub のリポジトリ名)
        * エンティティ： `branch`
        * GitHub 環境名： `main`
    * 資格情報
        * 名前： (任意)


## 作成したアプリにロールを付与

1. サブスクリプションを開く
1. [アクセス制御(IAM)]を開く
1. 「追加」→「ロールの割り当ての追加」を開く
1. ロールの割り当ての追加
    1. ロール
        * 「特権管理者ロール」タブへ移動して `共同作成者` を選択
    1. メンバー
        * 割当先： `ユーザー、グループ、またはサービスプリンシパル`
        * メンバー： (作成したアプリ)


## パイプラインの作成

```
jobs:
  build-deploy:
    ...
    steps:
      - name: Azure login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

`azure/login@v1` に与えるパラーメタ―は以下の通り

|パラメーター| 値 |
|---|---|
| `client-id` | AzureAD に登録したアプリの `アプリケーションID` |
| `tenant-id` | 作成した AzureAD アプリ の登録先 `テナントID` |
| `subscription-id` | 作成した AzureAD アプリ の登録先 `サブスクリプションID` |

