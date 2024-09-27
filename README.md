# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## Using deno

以下の方法を参考に実施する。
https://zenn.dev/methane/articles/2023-deno-zenn-cli

miseを使ってdenoを導入する。

```
mise use deno
```

denoを使って、zenn-cliをインストールする。

```
deno install -A npm:zenn-cli@latest
```

zenn-cliのPATHを、.envrcに追記する。

```
direnv edit .

export PATH="/Users/<username>/.local/share/mise/installs/deno/version/.deno/bin:$PATH"
```

実行できるかテストする。

```
zenn-cli --help
```
