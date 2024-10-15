# Python の静的型付け

> [!NOTE]
> このリポジトリは第三者による日本語訳です。 日本語訳は Python Software Foundation とは一切関係ありません。 このリポジトリは PSF ライセンスに準じて公開しています。

## ドキュメントとサポート

Python の静的型付けに関するドキュメントは [typing.readthedocs.io](https://typing.readthedocs.io/) にあります。サポートが必要な場合は、[サポートフォーラム](https://github.com/python/typing/discussions) をご利用ください。

型システムの改善については、[Python の Discourse](https://discuss.python.org/c/typing/32) で議論し、このリポジトリの [issues](https://github.com/python/typing/issues) で追跡されます。

チャットプラットフォームに適した会話については、以下のいずれかを使用できます：

- [gitter](https://gitter.im/python/typing)
- [discord](https://discord.com/channels/267624335836053506/891788761371906108) の `#type-hinting` チャンネル

## リポジトリの内容

この GitHub リポジトリは、いくつかの目的で使用されています：

- [typing.readthedocs.io](https://typing.readthedocs.io/) のドキュメントは [docs ディレクトリ](./docs) で管理されています。これには、型システムの[仕様](https://typing.readthedocs.io/en/latest/spec/index.html)が含まれます。特に、仕様の[更新手順](https://typing.readthedocs.io/en/latest/spec/meta.html)を参照してください。

- タイピング関連のユーザーヘルプのための[ディスカッションフォーラム](https://github.com/python/typing/discussions)がここにホストされています。

- Python 静的型チェッカーの[適合性テスト](https://github.com/python/typing/blob/main/conformance/README.md)。[最新の適合性テスト結果](https://htmlpreview.github.io/?https://github.com/python/typing/blob/main/conformance/results/results.html)はこちらです。

過去には、このリポジトリには以下もホストされていました：

- `typing_extensions` パッケージは現在 [typing_extensions](https://github.com/python/typing_extensions) リポジトリにあります。以前は `typing_extensions` ディレクトリにありました。

- 古い Python バージョン向けの [`typing` モジュール](https://docs.python.org/3/library/typing.html) のバックポート。`typing` が標準ライブラリに含まれていないすべての Python バージョンがサポート終了に達した後に削除されました。Python 2.7 および 3.4 をサポートする最後のリリースバージョンは [PyPI で利用可能](https://pypi.org/project/typing/) です。
