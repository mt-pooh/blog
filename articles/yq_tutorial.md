# CLIでyamlファイルをいい感じに操作できる「yq」を使ってみる

# モチベーション
最近 Kubernetes をいじり始めて yaml ファイルを扱うことが増えてきた。
そんな中， CI でマニフェストの image の tag 名を更新したくなり，いろいろ調べた結果，yq が良さそうということで使い方を調べてみた。

# yqとは
> yq is a lightweight and portable command-line YAML processor

> yq は軽量でポータブルなコマンドライン YAML プロセッサです。

https://github.com/mikefarah/yq
Doc: https://mikefarah.gitbook.io/yq/

要するにコマンドラインで yaml ファイルをいい感じに操作できるツールとのこと。
ちなみに，json をコマンドラインでいい感じに操作できるツール[jq](https://github.com/stedolan/jq)もあるらしい。

## install
https://mikefarah.gitbook.io/yq/#install
Mac

```
brew install yq
```
で OK。

ちなみに Docker image も配布している。
CI で使うことを考えると便利そうですね。
```
docker run --rm -v "${PWD}":/workdi mikefarah/yq yq [flags] <command> FILE...
```
# yq使ってみる
今回の目的は `postgres-pod.yaml` の image の値，`postgres`を `postgres:11` に書き換えることとする。

```
# postgres-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
  labels:
    name: postgres-pod
    app: demo-app
spec:
  containers:
    - name: postgres
      image: postgres
      ports:
        - containerPort: 5432
      env:
        - name: POSTGRES_USER
          value: "postgres"
        - name: POSTGRES_PASSWORD
          value: "postgres"
```

## 結論
```
yq write -i postgres-pod.yaml spec.containers[0].image postgres:11
```
で目的達成。簡単！

## 補足
### Path Expressions
yq 使う上で理解しておかなくてはいけないのが Path Expressions。
yaml の要素を指定する際に使う，表現方法です。
例えば `postgres-pod.yaml` の `image: postgres` を指定したい場合の Path Expression は
`spec.containers[0].image`
となる。詳しくは以下ドキュメントを参照。
https://mikefarah.gitbook.io/yq/usage/path-expressions

### `yq write`
```
yq write <yaml_file> <path_expression> <new value>
```
で使える。
よく使いそうなオプションは以下の通り。
`-i, --inplace `: yaml ファイルを直接更新する，付けない場合，標準出力に更新後の yaml の内容が出されるだけでファイル自体は更新しません。
` -v, --verbose`: 詳細を出力する，CI で実行させるときはつけると便利ですね。

# まとめ
yq を使えば CI でマニフェストファイルの更新が簡単にできそう。
`kubectl`を使わずに image tag を変更できるのはいい感じですね。
今回使った `yq write` 以外にも使い道はいろいろありそうなので，かつ[ドキュメント](https://mikefarah.gitbook.io/yq/)も充実しているので今後も使い続けたい。
