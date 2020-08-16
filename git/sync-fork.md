# フォーク元のリポジトリの更新を取り込む方法

## フォーク元リポジトリの設定

フォーク元のリポジトリを設定しているか確認する
```
❯ git remote -v
origin	ssh://git@github.com/bellwood4486/oapi-codegen.git (fetch)
origin	ssh://git@github.com/bellwood4486/oapi-codegen.git (push)
```

フォーク元のリポジトリを設定する
```
❯ git remote add upstream https://github.com/deepmap/oapi-codegen.git
```

こうなる
```
❯ git remote -v
origin	ssh://git@github.com/bellwood4486/oapi-codegen.git (fetch)
origin	ssh://git@github.com/bellwood4486/oapi-codegen.git (push)
upstream	https://github.com/deepmap/oapi-codegen.git (fetch)
upstream	https://github.com/deepmap/oapi-codegen.git (push)
```

## フォーク元の更新を反映する

```
❯ git fetch upstream
❯ git merge upstream/master --ff # デフォルトに--no-ffを設定している場合
```
