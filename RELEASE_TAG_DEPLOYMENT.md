# GitHubリリースタグによるデプロイメント機能

## 概要

このプロジェクトは、GitHubに「release」で始まるタグをプッシュすると、自動的にCI/CDパイプラインが起動するように設定されています。

## 🎯 主な変更点

### 1. CodeBuild Webhook設定
- GitHubのwebhookを有効化
- `release.*`パターンのタグがプッシュされた時のみビルドをトリガー
- 通常のコミット（main ブランチへのpush等）ではビルドは**トリガーされません**

### 2. Docker Image Tagging
- GitHubのタグ名がそのままDockerイメージのタグとして使用されます
- 例: `release-v1.0.0` タグをプッシュ → ECRに `release-v1.0.0` タグでイメージが登録
- 同時に `latest` タグでもイメージが登録されます

### 3. パイプライン実行フロー
```
GitHub Tag Push (release-*) 
  → CodeBuild Webhook トリガー
  → Docker イメージビルド & ECR Push
  → CodePipeline 起動
  → 手動承認待ち
  → ECS Fargate デプロイ
```

## 📝 使用方法

### デプロイの実行手順

#### 1. コードの変更をコミット
```bash
# 変更をコミット
git add .
git commit -m "新機能を追加"
git push origin main
```

#### 2. リリースタグを作成してプッシュ
```bash
# リリースタグを作成（必ず 'release' で始める必要があります）
git tag release-v1.0.0

# タグをGitHubにプッシュ
git push origin release-v1.0.0
```

#### 3. 自動でパイプラインが起動
- CodeBuildが自動的にトリガーされます
- AWSコンソールのCodeBuildで進捗を確認できます

#### 4. 手動承認を実施
- CodePipelineの「approve」ステージで手動承認が必要です
- AWSコンソールのCodePipelineで承認ボタンをクリック

#### 5. デプロイ完了
- ECS Fargateに新しいバージョンがデプロイされます

## 🏷️ タグ命名規則

以下のパターンのタグでパイプラインがトリガーされます:

✅ **有効なタグ名の例:**
- `release-v1.0.0`
- `release-v2.1.3`
- `release-hotfix-1.0.1`
- `release-2024.11.26`
- `release-production`

❌ **無効なタグ名の例（トリガーされません）:**
- `v1.0.0` （releaseで始まっていない）
- `rel-v1.0.0` （releaseで始まっていない）
- `release` （release単体はOKだが、バージョン番号を含めることを推奨）

## 🔍 トラブルシューティング

### タグをプッシュしてもビルドが起動しない

**確認事項:**
1. タグ名が `release` で始まっているか確認
   ```bash
   git tag  # ローカルのタグ一覧を表示
   ```

2. タグが正しくGitHubにプッシュされているか確認
   ```bash
   git ls-remote --tags origin  # リモートのタグ一覧を表示
   ```

3. CodeBuildのWebhook設定を確認
   - AWS Console → CodeBuild → プロジェクト → Webhooks で設定を確認

4. GitHub Personal Access Tokenの権限を確認
   - `admin:repo_hook` 権限が必要です

### CodeBuildは起動したがパイプラインが起動しない

**確認事項:**
1. CodeBuildのIAM Roleに `codepipeline:StartPipelineExecution` 権限があるか確認

2. CodeBuildのログを確認
   ```
   Starting pipeline execution...
   PIPELINE_NAME=$(aws codepipeline list-pipelines ...)
   ```
   このメッセージが表示されているか確認

### 特定のタグを削除したい

```bash
# ローカルのタグを削除
git tag -d release-v1.0.0

# リモート（GitHub）のタグを削除
git push origin :refs/tags/release-v1.0.0
```

## 📊 ECRイメージの確認

デプロイされたイメージは以下で確認できます:

```bash
# ECRリポジトリのイメージ一覧を表示
aws ecr list-images --repository-name <ECR-REPO-NAME>

# 特定のタグのイメージ詳細を表示
aws ecr describe-images --repository-name <ECR-REPO-NAME> --image-ids imageTag=release-v1.0.0
```

## 🔄 ロールバック方法

以前のバージョンにロールバックする場合:

### 方法1: 以前のタグを再デプロイ
```bash
# 同じタグ名は使用できないため、新しいタグ名で作成
git tag release-v1.0.0-rollback <古いコミットハッシュ>
git push origin release-v1.0.0-rollback
```

### 方法2: ECSコンソールから手動でロールバック
1. AWS Console → ECS → Service
2. 「Update Service」をクリック
3. Task Definition で以前のリビジョンを選択
4. 「Update」をクリック

## 📦 デプロイの再実行

既存のスタックを更新する場合:

```bash
cd cdk-v2
npm run build
cdk diff
cdk deploy --parameters githubUserName=<YOUR-GITHUB-USERNAME>
```

## 🔐 セキュリティ考慮事項

- GitHub Personal Access Tokenは AWS Secrets Manager で安全に管理されています
- CodeBuildは必要最小限の権限のみを持つIAM Roleを使用しています
- 手動承認ステップにより、意図しないデプロイを防止しています

## 📚 関連リソース

- [AWS CodeBuild - Webhooks](https://docs.aws.amazon.com/codebuild/latest/userguide/github-webhook.html)
- [AWS CodePipeline - Manual Approval](https://docs.aws.amazon.com/codepipeline/latest/userguide/approvals.html)
- [Amazon ECR - Image Tagging](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-tag-mutability.html)
