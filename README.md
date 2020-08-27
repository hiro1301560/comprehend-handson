# Amplify + Vue.js + Lambda + Comprehend で問合せした人の感情を判定するフォームを作る

こんにちは、JAWS-UG 浜松支部の松井です。  
近年、機械学習、AIに対する注目が集まっています。しかし、何らかの機械学習を活用した仕組みを1人で0から作るのはなかなかハードルが高く、機械学習で重要になってくるビッグデータの収集も大きな問題になってきます。  
AWSでは、こうした問題を解決してくれる、機械学習を誰でも簡単に活用できるサービスが多数提供されています。  
今回はその中でも、機械学習を使用してテキスト内でインサイトや関係性を検出する自然言語処理 (NLP) サービス[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を活用して、 **問合せした人の感情を判定してくれるフォーム** を作ってみようと思います。

- 機械学習、AI活用といっても、どこから手をつけたらいいか分からない
- 日常の生活や業務でどの様に役に立てれば良いか分からない

という方も多くいらっしゃると思いますが、[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を活用したこちらのハンズオンが一助となれば幸いです。

## 前提環境

今回のハンズオンは下記の環境&ツールにて検証しています。 ※必要なツールが揃っていればPCやOSは問いません

- OS: Mac OS X
- node: v14.6.0
- npm: 6.14.7
- vue: @vue/cli 4.4.6
- amplify: 4.27.3

また、[こちら](https://aws.amazon.com/jp/builders-flash/202008/amplify-crud-app/)をご参考に、上記の必要なツールを揃えた環境を[AWS Cloud9](https://aws.amazon.com/jp/cloud9/)上に構築していただくこともできます。

## 今回のアーキテクチャ

今回のハンズオンで作成するシステムの構成図です。

![aws_top](https://github.com/matsuihidetoshi/contact/blob/master/images/architecture.png)

- 動作の起点となるお問い合わせフォームは **Vue.js** を使って作成し、[AWS Amplify](https://aws.amazon.com/jp/amplify/)を使って[Amazon S3](https://aws.amazon.com/jp/s3/)バケット上にデプロイします。煩雑な環境構築のステップを省略し、証明書付きのURLを自動生成してくれるのでとても便利です。
- お問い合わせフォームの入力内容はAjaxを使ってPOSTリクエストを送信し、[Amazon API Gateway](https://aws.amazon.com/jp/api-gateway/)が受け取り、[AWS Lambda](https://aws.amazon.com/jp/lambda/)へプロキシします。
- [AWS Lambda](https://aws.amazon.com/jp/lambda/)では[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を使って問合せ内容を解析し、文章から感情を判定します。その判定内容を添えて、[Amazon SES](https://aws.amazon.com/jp/ses/)へ管理者へのメール送信をリクエストします。


## 1.SESの設定

[Amazon SES](https://aws.amazon.com/jp/ses/)を使って管理者（自分）にメールを送れるように設定を行います。

***

![ses_select](https://github.com/matsuihidetoshi/contact/blob/master/images/ses_select.png)

- [AWSマネジメントコンソール](https://console.aws.amazon.com/)にアクセスし、「ses」と入力し、検索結果（プルダウン）の「Simple Email Service」をクリックします。

***

![verify_email](https://github.com/matsuihidetoshi/contact/blob/master/images/verify_email.png)

- 左ペインの「Email Address」をクリックし、「Verify a New Mail Address」ボタンをクリックします。

***

![input_email](https://github.com/matsuihidetoshi/contact/blob/master/images/input_email.png)

- フォームにメールアドレスを入力し、「Verify This Email Address」ボタンをクリックします。※こちらの見本アドレスはテスト用です。ご自身の実在するメールアドレスを使用してください。

***

![click_link](https://github.com/matsuihidetoshi/contact/blob/master/images/click_link.png)

- 入力したメールアドレス宛に上記のようなメールが届くので、リンクをクリックしてメールアドレスの検証を完了させます。

## 2.Lambda関数の作成

[Amazon Comprehend](https://aws.amazon.com/jp/comprehend/)を使って送られてきた問合せ内容を解析して感情判定し、判定結果を加えた上で[Amazon SES](https://aws.amazon.com/jp/ses/)を使って管理者にメールを送信する関数を作成します。

***

![lambda_select](https://github.com/matsuihidetoshi/contact/blob/master/images/lambda_select.png)

- [AWSマネジメントコンソール](https://console.aws.amazon.com/)にアクセスし、「lambda」と入力し、検索結果（プルダウン）の「Lambda」をクリックします。

***

![lambda_create](https://github.com/matsuihidetoshi/contact/blob/master/images/lambda_create.png)

- 「関数の作成」をクリックします。

***

![lambda_config](https://github.com/matsuihidetoshi/contact/blob/master/images/lambda_config.png)

- 「関数名」に「contactFunction」と入力します（命名は何でも構いません）。
- 「ランタイム」に、今回は「Python 3.7」を選択します。
- 「実行ロールの選択または作成」の項目を展開し、「基本的なLambdaアクセス権限で新しいロールを作成」を選択します。
- 最後に、「関数の作成」をクリックします。

***

![lambda_role](https://github.com/matsuihidetoshi/contact/blob/master/images/lambda_role.png)

- 関数が正常に作成できたら、「アクセス権限」タブをクリックして、「ロール名」に表示されるロールをクリックします。

***

![policy_attach](https://github.com/matsuihidetoshi/contact/blob/master/images/policy_attach.png)

- 「ポリシーをアタッチします」ボタンをクリックします。

***

![comprehend_policy](https://github.com/matsuihidetoshi/contact/blob/master/images/comprehend_policy.png)

- 検索フォームに「comprehend」と入力します。
- 「ComprehendFullAccess」にチェックを入れます。
- 「ポリシーのアタッチ」ボタンをクリックします。

***

![ses_policy](https://github.com/matsuihidetoshi/contact/blob/master/images/ses_policy.png)

- 同じ要領で再度「ポリシーをアタッチします」ボタンを押してから、検索フォームに「ses」と入力します。
- 「AmazonSESFullAccess」にチェックを入れます。
- 「ポリシーのアタッチ」ボタンをクリックします。

***

![function_code](https://github.com/matsuihidetoshi/contact/blob/master/images/function_code.png)

- 作成したLambda関数「contactFunction」に戻り、「設定」タブの「関数コード」のエディタでコードを記述していきます。

lambda_function.py

```
import json
import boto3

REGION = 'ap-northeast-1'
SRC_MAIL = "test@example.com"#SESに登録したメールアドレス
DST_MAIL = "test@example.com"#SESに登録したメールアドレス
SENTIMENTS = {
    "NEUTRAL": "中立的",
    "POSITIVE": "肯定的",
    "NEGATIVE": "否定的",
    "MIXED": "混在"
}
LANGUAGE_CODE = 'ja'

def detect_text_sentiment(text):#Comprehendを使って感情を検知するメソッド
    comprehend = boto3.client('comprehend', region_name=REGION)
    response = comprehend.detect_sentiment(Text=text, LanguageCode=LANGUAGE_CODE)
    return response

def send_email(source, to, subject, body):#SESでメールを送るメソッド
    ses = boto3.client('ses', region_name=REGION)
 
    response = ses.send_email(
        Source=source,
        Destination={
            'ToAddresses': [
                to,
            ]
        },
        Message={
            'Subject': {
                'Data': subject,
            },
            'Body': {
                'Text': {
                    'Data': body,
                },
            }
        }
    )
     
    return response

def lambda_handler(event, context):
    sentiment = detect_text_sentiment(event['content'])['Sentiment']
    body = "件名:\n" + event['title'] + "\n" + "\n連絡先:\n" + event['contact'] + "\n" + "\n感情:\n" + SENTIMENTS[sentiment] + "\n" + "\n内容:\n" + event['content']
    mail = send_email(SRC_MAIL, DST_MAIL, 'お問い合わせがありました', body)
    
    return {
        'statusCode': 200,
        'body': {
            'sentiment': sentiment,
            'mail': mail
        }
    }
```

- 上記の通りのコードを記述します。

***

![create_test](https://github.com/matsuihidetoshi/contact/blob/master/images/create_test.png)

- 関数のテストイベントを作成して、動作確認します。
- 上記の「テスト」ボタンをクリックします。

***

![configure_test](https://github.com/matsuihidetoshi/contact/blob/master/images/configure_test.png)

- 上記の通り、「イベント名」（任意ですのでなんでも構いません）とテスト用のパラメータを記述して、「作成」ボタンをクリックします。

***

![create_test](https://github.com/matsuihidetoshi/contact/blob/master/images/create_test.png)

- もう一度「テスト」ボタンをクリックすると、テストが実行されます。

***

![mail_result](https://github.com/matsuihidetoshi/contact/blob/master/images/mail_result.png)

- 登録したアドレスに、上記の通りのメールが送られていれば成功です。
- 「感情」という項目に、感情の判定結果が表示されているかと思います。

## 3.API Gatewayの設定

作成したLambda関数をAPIとして呼び出しできる様に、[Amazon API Gateway](https://aws.amazon.com/jp/api-gateway/)を設定していきます。

***

![add_trigger](https://github.com/matsuihidetoshi/contact/blob/master/images/add_trigger.png)

- 引き続き作成したLambda関数「contactFunction」の画面で、「＋トリガーを追加」ボタンをクリックします。

***

![configure_trigger](https://github.com/matsuihidetoshi/contact/blob/master/images/configure_trigger.png)

- 選択肢はそれぞれ「API Gateway」「Create an API」「REST API」「オープン」を選択し、「追加」ボタンをクリックします。

***

![select_trigger](https://github.com/matsuihidetoshi/contact/blob/master/images/select_trigger.png)

- Lambda関数の画面に戻るので、「contactFunction-API」をクリックします。

***

![create_method](https://github.com/matsuihidetoshi/contact/blob/master/images/create_method.png)

- 「アクション」をクリックし、プルダウンメニューから「メソッドの作成」をクリックします。

***

![select_post](https://github.com/matsuihidetoshi/contact/blob/master/images/select_post.png)

![method_check](https://github.com/matsuihidetoshi/contact/blob/master/images/method_check.png)

- 表示されるプルダウンメニューの「POST」をクリックして、右側のチェックマークをクリックします。

***

![method_setup](https://github.com/matsuihidetoshi/contact/blob/master/images/method_setup.png)

- その後表示されるメソッドのセットアップ画面で、Lambda関数の項目で「contactFunction(作成したLambda関数名)」を入力（入力中に候補が表示されるので、クリックでOKです）します。
- 「保存」ボタンをクリックします。

***

![method_setup_ok](https://github.com/matsuihidetoshi/contact/blob/master/images/method_setup_ok.png)

- 「OK」ボタンをクリックします。

***

![integrated_request](https://github.com/matsuihidetoshi/contact/blob/master/images/integrated_request.png)

- 続いて、「統合リクエスト」をクリックします。

***

![add_mapping_template](https://github.com/matsuihidetoshi/contact/blob/master/images/add_mapping_template.png)

- 「マッピングテンプレート」をクリックし、「テンプレートが定義されていない場合 (推奨) 」を選択し、「マッピングテンプレートの追加」をクリックします。

***

![open_mapping_template](https://github.com/matsuihidetoshi/contact/blob/master/images/open_mapping_template.png)

- 「application/json」と入力し、チェックマークをクリックします。

***

![enter_mapping_template](https://github.com/matsuihidetoshi/contact/blob/master/images/enter_mapping_template.png)

- 下にスクロールして、マッピングテンプレートの入力フィールドをクリックします。

```
{
  "contact": $input.json("$.contact"),
  "title": $input.json("$.title"),
  "content": $input.json("$.content")
}
```

- 上記の通りのテンプレートを入力して、「保存」をクリックします。

***

![select_cors](https://github.com/matsuihidetoshi/contact/blob/master/images/select_cors.png)

- 「アクション」から「CORS の有効化」をクリックします。

***

![activate_cors](https://github.com/matsuihidetoshi/contact/blob/master/images/activate_cors.png)

- 「CORS を有効にして既存の CORS ヘッダーを置換」ボタンをクリックします。

***

![confirm_cors](https://github.com/matsuihidetoshi/contact/blob/master/images/confirm_cors.png)

- 「はい、既存の値を置き換えます」ボタンをクリックします。

***

![select_api_deploy](https://github.com/matsuihidetoshi/contact/blob/master/images/select_api_deploy.png)

- 「アクション」から「API のデプロイ」をクリックします。

***

![confirm_api_deploy](https://github.com/matsuihidetoshi/contact/blob/master/images/confirm_api_deploy.png)

- 「デプロイされるステージ」に「default」を選択し、「デプロイ」をクリックします。

***

![test_post](https://github.com/matsuihidetoshi/contact/blob/master/images/test_post.png)

- 次のページの上記に表示されるURLの末尾に「/contactFunction」を追加したURLに対して、**curl**でPOSTメソッドのリクエストを送信して、動作確認しましょう。

Terminal

```
curl -X POST -H "Content-Type: application/json" -d '{"title":"curl", "contact":"curl@example.com", "content": "I am so happy."}' https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/default/contactFunction
```

- ターミナルにて、上記コマンドを実行して、登録したアドレスに正常にメールが送信されればOKです。

***

## 3.Vue.jsプロジェクトを作成

Vue CLIを使って、必要な設定を行いながらVue.jsプロジェクトを作成していきます。

#### vue create

notesという名前でプロジェクトを作成

```
vue create notes
```

#### npmリポジトリの設定(y or nを入力しEnter)

nを入力しEnter

```
?  Your connection to the default npm registry seems to be slow.
   Use https://registry.npm.taobao.org for faster installation? (Y/n) n
```

#### 設定方法の選択(↑↓でカーソル移動、Enterで決定)

Manually select featuresを選択しEnter

```
Vue CLI v4.1.2
? Please pick a preset: 
  default (babel, eslint) 
❯ Manually select features 
```

#### 追加パッケージを選択(↑↓でカーソル移動、Spaceで選択、Enterで決定)

下記の通り、Babel, Progressive Web App (PWA) Support, Router, Vuex, Linter / Formatterを選択

```
? Check the features needed for your project: 
 ◉ Babel
 ◯ TypeScript
 ◉ Progressive Web App (PWA) Support
 ◉ Router
❯◉ Vuex
 ◯ CSS Pre-processors
 ◉ Linter / Formatter
 ◯ Unit Testing
 ◯ E2E Testing
 ```
 
 #### Historyモードを選択(y or nを入力しEnter)
 
 yを入力しEnter
 
 ```
 ? Use history mode for router? (Requires proper server setup for index fallback in production) (Y/n) y
 ```
 
 #### ESLintの設定を選択(↑↓でカーソル移動、Enterで決定)
 
 デフォルトのESLint with error prevention onlyを選択しEnter
 
 ```
 ? Pick a linter / formatter config: (Use arrow keys)
❯ ESLint with error prevention only 
  ESLint + Airbnb config 
  ESLint + Standard config 
  ESLint + Prettier 
```

#### Lintのタイミングの設定(↑↓でカーソル移動、Enterで決定)

デフォルトのLint on saveを選択しEnter

```
? Pick additional lint features: (Press <space> to select, <a> to toggle all, <i> to invert selection)
❯◉ Lint on save
 ◯ Lint and fix on commit
 ```
 
#### Configファイルの設定(↑↓でカーソル移動、Enterで決定)

デフォルトのIn dedicated config filesを選択しEnter

```
? Where do you prefer placing config for Babel, ESLint, etc.? (Use arrow keys)
❯ In dedicated config files 
  In package.json
```

#### 今回の設定を保存指定次回以降に転用するかの選択(y or nを入力しEnter)

任意だが、今回はnを入力しEnter

```
? Save this as a preset for future projects? (y/N) n
```

## Vue.jsプロジェクトにCloud9向けの設定を追加

このままでは、Cloud9の仕組み上、起動した開発環境のサーバーをうまくプレビューできないため  
設定を追加します。

#### 設定ファイルの追加

- vueamplifydev/vue.config.js

を

- notes/vue.config.js

となる様にコピー

## テスト起動

開発環境のローカルサーバーを起動してプレビューし、問題なくVue.jsプロジェクトが  
立ち上がることを確認します。

#### Vue.jsプロジェクトフォルダ(notes)に移動

```
cd notes
```

#### サーバー起動

```
npm run serve
```

Cloud9画面の上部のPreview→Preview Running Applicationを、Vue.jsのデフォルト画面が表示されればOK  
*そのままサーバーは起動したまま、別ターミナルを開き、notesディレクトリに移動して以降の作業を実施

## Amplify初期設定

Amplify CLIを使用して各種機能追加に伴うコードの生成やAWSリソースの追加を自動で行うことができます。  
AWSのリソースの操作をCLIから行える様にするために、専用のIAMユーザーの追加と認証情報の設定を行う必要があります。

#### Amplify CLIから機能追加・各リソースの構築ができる様に設定

```
amplify configure
```

#### AWSマネジメントコンソールへのリンクが表示されるが、そのままEnter

```
Follow these steps to set up access to your AWS account:

Sign in to your AWS administrator account:
https://console.aws.amazon.com/
Press Enter to continue
```

#### AWSリソースのリージョン選択(↑↓でカーソル移動、Enterで決定)

下記の通りap-northeast-1を選択しEnter

```
Specify the AWS Region
? region:  
  eu-west-1 
  eu-west-2 
  eu-central-1 
❯ ap-northeast-1 
  ap-northeast-2 
  ap-southeast-1 
  ap-southeast-2 
(Move up and down to reveal more choices)
```

#### Amplify用IAMユーザーの追加(ユーザー名を入力しEnter)

任意だが今回はデフォルトのユーザー名を使用(そのままEnter)

```
Specify the username of the new IAM user:
? user name:  (amplify-xxxxx) 
```

#### IAMユーザー作成画面を開く

下記の様に案内されるURLを開く

```
Complete the user creation using the AWS console
https://console.aws.amazon.com/iam/home?region=undefined#/users$new?step=final&accessKey&userNames=amplify-xxxxx&permissionType=policies&policies=arn:aws:iam::aws:policy%2FAdministratorAccess
Press Enter to continue
```

#### IAMユーザー作成

ブラウザの別タブでAWSマネジメントコンソールのIAMユーザー作成画面が開くので  
ユーザー詳細の設定→アクセス許可の設定→タグの追加 (オプション)→確認まで全てデフォルトのまま進み  
「ユーザーの作成」ボタンを押下

#### IAMユーザーセキュリティ認証情報の取得

IAMユーザーセキュリティ認証情報のページが開くので、「.csvのダウンロード」ボタンを押下  
*このあとすぐダウンロードしたファイル内の情報を使用するため、ブラウザ下部のダウンロードファイルの表示を  
開いたままにするなどし、すぐに開ける様にしておく

#### ターミナル操作に戻る

Cloud9環境の画面に戻る

#### IAMユーザー作成完了

下記の表示の状態で、Enter

```
Complete the user creation using the AWS console
https://console.aws.amazon.com/iam/home?region=undefined#/users$new?step=final&accessKey&userNames=amplify-xxxxx&permissionType=policies&policies=arn:aws:iam::aws:policy%2FAdministratorAccess
Press Enter to continue
```

#### アクセスキーIDの入力(アクセスキーIDを入力しEnter)

先ほどダウンロードした.csvファイルをブラウザ下部から開き、Access key IDをコピー  
ターミナルに戻り、ペーストしEnter

```
Enter the access key of the newly created user:
? accessKeyId:  (<YOUR_ACCESS_KEY_ID>) 
```

#### シークレットアクセスキーを入力(シークレットアクセスキーを入力しEnter)

同じく.csvファイルよりSecret access keyをコピー  
ターミナルに戻り、ペーストしEnter

```
? secretAccessKey:  (<YOUR_SECRET_ACCESS_KEY>) 
```

#### プロファイルに名前をつける(プロファイル名を入力しEnter)

デフォルトのままEnter

```
Profile Name:  (default)
```

*ここで、AWS managed temporary credentialsというダイアログが立ち上がり  
Could not update credentialsと表示されるが、「Fource update」ボタンを押下すればOK  
ターミナルは下記のメッセージが出力されていればOK

```
Successfully set up the new user.
```

## Vue.jsプロジェクトをAmplifyプロジェクトとして初期化

Vue.jsプロジェクトをAmplifyプロジェクトとして初期化することで、各種機能追加に伴うコードの自動生成と  
プロジェクトのフロントエンドに紐付いたバックエンドの役割を担う各種AWSリソースの構築を  
CLIから実行することができる様になります。  
この操作が完了すると、同時にAmplifyバックエンドの素体の様なものがCloudFormationによりクラウド上に作成されます。

```
amplify init
```

#### プロジェクト名の設定(プロジェクト名を入力しEnter)

デフォルト(Vue.jsプロジェクトとしての名前:notes)のままEnter

```
Note: It is recommended to run this command from the root of your app directory
? Enter a name for the project (notes)
```

#### 環境名の設定(環境名を入力しEnter)

任意だが、今回はdefaultと入力しEnter

```
? Enter a name for the environment default
```

#### エディタの設定(↑↓でカーソル移動、Enterで決定)

Noneを選択しEnter

```
? Choose your default editor: 
  Visual Studio Code 
  Atom Editor 
  Sublime Text 
  IntelliJ IDEA 
  Vim (via Terminal, Mac OS only) 
  Emacs (via Terminal, Mac OS only) 
❯ None
```

#### アプリケーションのタイプを選択(↑↓でカーソル移動、Enterで決定)

javascriptを選択しEnter

```
? Choose the type of app that you're building (Use arrow keys)
  android 
  ios 
❯ javascript
```

#### フレームワークを選択(↑↓でカーソル移動、Enterで決定)

vueを選択しEnter

```
? What javascript framework are you using (Use arrow keys)
  angular 
  ember 
  ionic 
  react 
  react-native 
❯ vue 
  none
```

#### ソースディレクトリを指定(ソースディレクトリ名を入力しEnter)

デフォルト(src)のままEnter

```
? Source Directory Path:  (src)
```

#### ディストリビューションディレクトリを指定(ディストリビューションディレクトリ名を入力しEnter)

デフォルト(dist)のままEnter

```
? Distribution Directory Path: (dist)
```

#### ビルドコマンドの入力(ビルドコマンドを入力しEnter)

npm run buildと入力しEnter

```
? Build Command:  npm run build
```

#### サーバー起動コマンドを入力(サーバー起動コマンドを入力しEnter)

npm run serveと入力しEnter

```
? Start Command: npm run serve
```

#### AWSプロファイルの使用(y or nを入力しEnter)

yを入力しEnter

```
Using default provider  awscloudformation

For more information on AWS Profiles, see:
https://docs.aws.amazon.com/cli/latest/userguide/cli-multiple-profiles.html

? Do you want to use an AWS profile? (Y/n) y
```

#### AWSプロファイルを選択(↑↓でカーソル移動、Enterで決定)

先ほど作成したプロファイル(default)を選択しEnter

```
? Please choose the profile you want to use (Use arrow keys)
❯ default
```

下記の様なメッセージが表示されれば成功

```
✔ Successfully created initial AWS cloud resources for deployments.
✔ Initialized provider successfully.
Initialized your environment successfully.

Your project has been successfully initialized and connected to the cloud!

Some next steps:
"amplify status" will show you what you've added already and if it's locally configured or deployed
"amplify add <category>" will allow you to add features like user login or a backend API
"amplify push" will build all your local backend resources and provision it in the cloud
“amplify console” to open the Amplify Console and view your project status
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud

Pro tip:
Try "amplify add api" to create a backend API and then "amplify publish" to deploy everything
```

## 認証機能の追加

アプリケーションに認証(登録・メール認証・サインイン・サインアウト・パスワードリセット）機能を追加します。  
Amplifyを導入しているため、コマンドを実行することで最低限必要なフロントエンドのコードの生成と
バックエンドの認証基盤(Lambda + Cognito)の構築を数ステップで完了できます。

#### 認証機能追加コマンド実行

```
amplify add auth
```

#### 設定タイプの選択(↑↓でカーソル移動、Enterで決定)

Default configurationを選択しEnter

```
Do you want to use the default authentication and security configuration? (Us
e arrow keys)
❯ Default configuration 
  Default configuration with Social Provider (Federation) 
  Manual configuration 
  I want to learn more. 
```

#### 認証に使う情報を選択(↑↓でカーソル移動、Enterで決定)

Emailを選択しEnter

```
 Warning: you will not be able to edit these selections. 
 How do you want users to be able to sign in? 
  Username 
❯ Email 
  Phone Number 
  Email and Phone Number 
  I want to learn more. 
```

#### 追加の設定に関する選択(↑↓でカーソル移動、Enterで決定)

No, I am done.を選択しEnter

```
 Do you want to configure advanced settings? (Use arrow keys)
❯ No, I am done. 
  Yes, I want to make some additional changes.
```

## APIの追加

アプリケーションに基本的なAPIを追加します。  
認証機能と同様、コマンドで簡単にフロントエンド側のコードと、CRUDが可能なAPIバックエンド  
(AppSync + DynamoDB)を構築できます。  
また、GraphQLのSubscriptionにより、Websocketを使ったイベントドリブンなフロントエンドへの  
リアルタイムデータ更新を実現します。

#### API追加コマンド実行

```
amplify add api
```

#### APIのインターフェースを選択(↑↓でカーソル移動、Enterで決定)

GraphQLを選択しEnter

```
? Please select from one of the below mentioned services: (Use arrow keys)
❯ GraphQL 
  REST 
```

#### APIに名前をつける(API名を入力しEnter)

デフォルトのアプリケーション名(notes)のままEnter

```
? Provide API name: (notes)
```

#### APIの認証タイプを選択(↑↓でカーソル移動、Enterで決定)

Amazon Cognito User Poolを選択しEnter

```
? Choose the default authorization type for the API 
  API key 
❯ Amazon Cognito User Pool 
  IAM 
  OpenID Connect
```

#### 追加の設定に関する選択(↑↓でカーソル移動、Enterで決定)

No, I am done.を選択しEnter

```
? Do you want to configure advanced settings for the GraphQL API (Use arrow ke
ys)
❯ No, I am done. 
  Yes, I want to make some additional changes.
```

#### GraphQLのスキーマがあるかの確認(y or nを入力しEnter)

nを入力しEnter

```
? Do you have an annotated GraphQL schema? (y/N) n
```

#### GraphQLの提携のスキーマを生成するかの確認(y or nを入力しEnter)

yを入力しEnter

```
? Do you want a guided schema creation? (Y/n) y
```

#### GraphQLのスキーマの雛形を選択(↑↓でカーソル移動、Enterで決定)

Objects with fine-grained access control  
(e.g., a project management app with owner-based authorization)を選択しEnter

```
? What best describes your project: 
  Single object with fields (e.g., “Todo” with ID, name, description) 
  One-to-many relationship (e.g., “Blogs” with “Posts” and “Comments”) 
❯ Objects with fine-grained access control (e.g., a project management app
  with owner-based authorization)
```

#### 今すぐGraphQLスキーマを編集するかの確認(y or nを入力しEnter)

yを入力しEnter

```
? Do you want to edit the schema now? (Y/n) y
```

下記の様なメッセージが表示されればOK

```
GraphQL schema compiled successfully.

Edit your schema at /home/ec2-user/environment/notes/amplify/backend/api/notes/schema.graphql or place .graphql files in a directory at /home/ec2-user/environment/notes/amplify/backend/api/notes/schema
Successfully added resource notes locally

Some next steps:
"amplify push" will build all your local backend resources and provision it in the cloud
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud
```

#### GraphQLスキーマの変更

下記の通り、/notes/amplify/backend/api/notes/schema.graphqlのファイルを編集

```
type Task 
  @model 
  @auth(rules: [
      {allow: groups, groups: ["Managers"], queries: null, mutations: [create, update, delete]},
      {allow: groups, groups: ["Employees"], queries: [get, list], mutations: null}
    ])
{
  id: ID!
  title: String!
  description: String
  status: String
}
type PrivateNote
  @model
  @auth(rules: [{allow: owner}])
{
  id: ID!
  content: String!
  updatedAt: String       #この1行のみ追加
}
```

## バックエンドのデプロイ

現状では、バックエンドの構築と連携を実現するのに最低限必要なローカル環境のコードを自動生成しただけでした。  
ここでは、生成されたコードに基づき、実際にバックエンド環境をAWSクラウド上に構築していきます。

#### バックエンドのデプロイコマンドを実行

```
amplify push
```

#### デプロイする内容の確認(y or nを入力しEnter)

yを入力しEnter

```
Scanning for plugins...
Plugin scan successful
✔ Successfully pulled backend environment default from the cloud.

Current Environment: default

| Category | Resource name | Operation | Provider plugin   |
| -------- | ------------- | --------- | ----------------- |
| Auth     | notes694e6855 | Create    | awscloudformation |
| Api      | notes         | Create    | awscloudformation |
? Are you sure you want to continue? (Y/n) y
```

#### GraphQLを操作するコードを自動生成するかの確認(y or nを入力しEnter)

yを入力しEnter

```
GraphQL schema compiled successfully.

Edit your schema at /home/ec2-user/environment/notes/amplify/backend/api/notes/schema.graphql or place .graphql files in a directory at /home/ec2-user/environment/notes/amplify/backend/api/notes/schema
? Do you want to generate code for your newly created GraphQL API (Y/n) y
```

#### 自動生成するGraphQLコードの言語の選択(↑↓でカーソル移動、Enterで決定)

javascriptを選択しEnter

```
? Choose the code generation language target (Use arrow keys)
❯ javascript 
  typescript 
  flow 
```

#### 自動生成するGraphQLのコードの階層構造を指定(階層構造を入力しEnter)

デフォルト(src/graphql/**/*.js)のままEnter

```
? Enter the file name pattern of graphql queries, mutations and subscriptions (src/
graphql/**/*.js)
```

#### 全てのタイプのGraphQLの操作コードを自動生成するかの確認(y or nを入力しEnter)

yを入力しEnter

```
? Do you want to generate/update all possible GraphQL operations - queries, mutatio
ns and subscriptions (Y/n) y
```

#### GraphQLの各リソースの相互参照の深さを指定(数値を入力しEnter)

デフォルト(2)のままEnter

```
? Enter maximum statement depth [increase from default if your schema is deeply nested] (2)
```

下記の様なメッセージが表示されればOK

```
✔ Generated GraphQL operations successfully and saved at src/graphql
✔ All resources are updated in the cloud

GraphQL endpoint: https://xxxxxxxxxxxxxxxxxxxxxxxxxx.appsync-api.ap-northeast-1.amazonaws.com/graphql

ec2-user:~/environment/notes (master)
```

## フロントエンド側のコードを作成

ここまでで、  

- 認証
- API

という二つの機能の最低限のコードと、バックエンドの環境が生成されました。  
ここから、フロントエンドで実際に機能するコードを作成していきます。  
今回は、すでにこのリポジトリにコードがあるのでそれをコピーしていきます。

#### 必要パッケージのインストール

```
npm i aws-amplify aws-amplify-vue lodash
```

#### localise.js

ログイン画面を日本語メッセージにするための設定を記述したファイルです

- vueamplifydev/src/localize.js

を

- notes/src/localize.js

となる様にコピー

#### SignIn.vue

認証機能を提供するコンポーネントです

- vueamplifydev/src/components/SignIn.vue

を

- notes/src/components/SignIn.vue

となる様にコピー

#### PrivateNote.vue

今回の主要な機能であるメモ機能を提供するコンポーネントです

- vueamplifydev/src/components/PrivateNote.vue

を

- notes/src/components/PrivateNote.vue

となる様にコピー

#### main.js

アプリケーションのエントリーポイントになるファイルです。  
Amplify関連のパッケージを導入するために記述を追加します。

- vueamplifydev/src/main.js

の内容を

- notes/src/main.js

に上書き

#### router/index.js

ルーティングの設定を記述するファイルです。  
ログイン状態を管理してページのアクセス制御をします。

- vueamplifydev/src/router/index.js

の内容を

- notes/src/router/index.js

に上書き

#### App.vue

アプリケーションの共通レイアウトのコンポーネントです。  
各ページへのナビゲーションを追加します。

- vueamplifydev/src/App.vue

の内容を

- notes/src/App.vue

に上書き

#### 動作確認

すでにアプリケーションがローカルで起動している状態だったら、プレビューしてみてください。  
起動していなければ、別ターミナルを開いて、

```
npm run serve
```

を実行してアプリケーションを起動し、プレビューしてみてください。  
正しくメモを投稿でき、リアルタイムに表示が更新されればOKです。

## フロントエンドのデプロイ

ここまでで、

- バックエンドの本番環境

- フロントエンドのコード

が完成しました。あとは、フロントエンドの本番環境のホスティングができれば完成です。

#### GitHubにコードをプッシュ

まず、今までの変更をGitでコミットしていきます。

```
git add .
git commit -m 'built application'
```

その後、GitHubにログインしてください。  
その後、上部ナビゲーションバー右上の`+`をクリックし、`New repository`をクリックしてください。  
`Repository name`に、`notes`と入力し、それ以外はデフォルトのまま`Create repository`をクリックしてください。  
そのあとリダイレクトした画面で`…or push an existing repository from the command line`という項目があります。  
その内容をコピーし、ターミナルでプロジェクトフォルダにて実行してください。

```
git remote add origin https://github.com/ユーザー名/notes.git
git push -u origin master
```

下記のようにユーザー名、パスワードを求められるので、GitHubの`ユーザー名` `パスワード`を入力してください。

```
Username for 'https://github.com/matsuihidetoshi/notes-final.git':ユーザー名
Password for 'https://matsuihidetoshi@github.com/matsuihidetoshi/notes-final.git':パスワード
```

これで、GitHubにコードがプッシュされ、デプロイの準備ができました。

#### Amplifyコンソールからデプロイ

通常、Webページをホスティングするには、サーバー構築・ネットワーク設定・ミドルウェアのインストール及び設定等が必要ですが、  
Amplifyコンソールを使用するとWebインターフェースから少ないステップで簡単にデプロイできます。

#### Amplifyコンソールを開く

まず、AWSマネジメントコンソールから、`Amplify`を検索し、選択します。  
Amplifyコンソールが開きますが、すでに`notes`というアプリケーションの項目が作成されているはずですので、それをクリックしてください。  

#### デプロイ - Githubの連携

フロントエンドのコードとして、どのGitリポジトリを参照するかの選択するかの画面が表示されますので、`GitHub`を選択し、`Connect branch`をクリックしてください。  
  
GitHubの認証ページが開きますので、`ユーザー名` `パスワード`を入力してログインしてください。  
OAuthによるアクセス許可の画面が表示されますので、`Authorize xxxx`をクリックしてください。
  
`GitHub 認証が成功しました。`と表示されます。  
下部の`リポジトリ`にて、`notes`を選択し、ブランチは`master`を選択し、`次へ`をクリックしてください。  

#### デプロイ - ビルド設定

`ビルド設定の構成`画面が開くので、`Select a backend environment`で`default`を選択してください。

#### デプロイ - ロールの作成

`Select an existing service role or create a new one so Amplify Console may access your resources.`という項目で、  
`Create new role`をクリックしてください。  
  
`ロールの作成`画面が表示されますので、デフォルトのまま`次のステップ: アクセス権限`をクリックしてください。
  
次の画面で`Attached アクセス権限ポリシー`という項目などが表示されますが、こちらもデフォルトのまま`次のステップ: タグ`をクリックしてください。  
  
次の画面で`タグの追加（オプション）`という項目が表示されますが、こちらもデフォルトのまま`次のステップ: 確認`をクリックしてください。  
  
確認画面が開きますが、そのまま`ロールの作成`をクリックしてください。  
画面が遷移したら、そのページは閉じてしまって構いません。  

#### デプロイ- ビルド設定2

先ほど開いていたAmplifyコンソールに戻り、  
`Select an existing service role or create a new one so Amplify Console may access your resources.`の項目のプルダウンの横の🔄マークをクリックし、  
先ほど作成した`amplifyconsole-backend-role`を選択してください。  
  
その後、`次へ`をクリックしてください。  
  
`確認`画面が表示されますが、`保存してデプロイ`をクリックしてください。  
  
ここから少し時間がかかりますが（10分ほど）、デプロイのフローが終わるまでお待ちください。
  
デプロイが完了すると、`ドメイン`という項目ができていますので、そちらのリンクをクリックすると、アプリケーションが開きます。  

## アプリケーションの削除

デプロイしたアプリケーションは、そのままにしておくとオンデマンドで**課金**されます。  
よほどアクセスが集中しない限り、ほとんど無視できるレベルの金額しか課金されませんが、  
全てのリソースを削除しておけば課金されることはありません。  
以下にアプリケーションリソース全体の削除方法を記載します。

#### CLIによる削除

まずはバックエンドのリソースを削除します。  
プロジェクトフォルダにて、

```
amplify delete
```

を実行してください。  
すると下記の通り確認が表示されますので、承諾してください(yを入力しEnter)。

```
? Are you sure you want to continue? (This would delete all the environments of 
the project from the cloud and wipe out all the local amplify resource files) (Y/n) y
```

問題なく削除されれば、下記の通り表示されます。

```
✔ Project deleted in the cloud
Project deleted locally.
```

#### Amplifyコンソールから削除

S3にデプロイされたアプリケーションのフロントエンドも削除する必要があります。  
**Amplifyコンソール**を開き、**全てのアプリ**→**notes**を選択してください。  
画面右上の**アクション**から、**アプリの削除**を選択してください。  
確認用ダイアログが表示されるので、フォームに**delete**を入力し、**Delete**を押下してください。

これで、今回のハンズオンで作成したアプリケーションの全てのリソースが削除されます。  
**今回使用したCloud9環境を使用しない場合、別途削除していただく必要があります  
(使用する分だけEC2インスタンスタイプに応じた課金が発生します)。**

  
以上で終了です。お疲れ様でした！
