
作業する時は
git fetch upstream
git switch -c 作業ブランチ --no-track upstream/main
でブランチを切って行う

--no-track を付けると追跡先が設定されないため、素の git push が origin に向かいます。
付けない場合は初回だけ git push -u origin 作業名 としてください。

