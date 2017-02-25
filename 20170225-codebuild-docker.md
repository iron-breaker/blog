AWS CodeBuildを使ってDockerイメージをビルドし、Amazon EC2 Container Registry(ECR)へpushする


コンニチハ、千葉です。

CodeBuildにて、Dockerをビルドしてみたのでご紹介です。

GitHub > CodeBuild > ECR
  - Dockerfile
  - buildspec.yml

GitHubに配置しているDockerfileを元にビルドし、コンテナイメージをECRにpushします。
pushしたコンテナイメージは、ローカル端末にダウンロードして起動し確認してみます。

## 作業サマリ

作業サマリです。

- ECRリポジトリを作成
- CodeBuild用のIAMロール作成
- Codebuildプロジェクト作成
- gitへコードをpush
- ビルド
- pushしたコンテナイメージをローカル端末で起動してみる

## やってみた
では、さっそくやっていきます。

### ECRリポジトリを作成する
マネージメントコンソールの、ECSよりリポジトリを作成します。

★０

### CodeBuild用のIAMロールを作成する

CodeBuildで利用するIAMロールを作成します。
IAMロールを作成して、ポリシーをアタッチします。

#### CodeBuild操作用のポリシー
ハイライト部分は適宜修正してください。

[code highligh="7,8"]
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Resource": [
                "arn:aws:logs:us-east-1:xxxxxxxxxxx:log-group:/aws/codebuild/build-docker",
                "arn:aws:logs:us-east-1:xxxxxxxxxxx:log-group:/aws/codebuild/build-docker:*"
            ],
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ]
        },
        {
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::codepipeline-us-east-1-*"
            ],
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:GetObjectVersion"
            ]
        },
        {
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:CompleteLayerUpload",
                "ecr:GetAuthorizationToken",
                "ecr:InitiateLayerUpload",
                "ecr:PutImage",
                "ecr:UploadLayerPart"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
[/code]

### Codebuildプロジェクト作成
マネージメントコンソールから、CodeBuildを選択し「Create Project」からプロジェクトを作成します。

★１

以下、それぞれ指定しました
- ソースの配置先※今回はGithub
- ランタイムとしてDockerを指定
- アーティファクトはなし

また「Show advanced settings」より、下記のように環境変数を指定します。
リージョン、アカウントID、ECRリポジトリを指定します。

★２

"name": "AWS_DEFAULT_REGION"
"value": "region-ID"

"name": "AWS_ACCOUNT_ID"
"value": "account-ID"

"name": "IMAGE_REPO_NAME"
"value": "Amazon-ECR-repo-name"

"name": "IMAGE_TAG"
"value": "latest"

### GitHubへコードをpush

今回は、リポジトリとしてGitHubを選択しました。以下のファイルをpushしました。

#### ファイルの配置
[code]
├── Dockerfile
└── buildspec.yml
[/code]

#### Dockerfile
[code]
FROM maven:3.3.9-jdk-8

RUN echo "Hello World"
[/code]

#### buildspec.yml
[code]
version: 0.1

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $IMAGE_REPO_NAME .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
[/code]

## ビルド

マネージメントコンソールのCodeBuildより、「Start build」をクリックします。

★４

ビルドが成功し、ECRにイメージがアップされます。

★５

## 最後に
簡単に設定できました。AWS内で完結するので、外部サービスとのキーのやりとりもないので、よりセキュアに利用できそうです。

# 参考
http://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html
