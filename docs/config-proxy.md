# プロキシ設定いろいろ

`proxy.example.com:80` は必要に応じて変更すること！！

`.bachrc`だったかに書いてもプロキシを有効にしてくれないことが何度かあったので気を付けて

## pip
```
pip install hoge --proxy=proxy.example.com:80
```
## Python
```
python get-pip.py --proxy=http://proxy.example.com:80
```
