******************************************************************************************
Python の静的型付け
******************************************************************************************

ガイド
==========================================================================================

.. toctree::
   :maxdepth: 2

   guides/index

リファレンス
==========================================================================================

.. toctree::
   :maxdepth: 2

   reference/index

.. seealso::

   https://mypy.readthedocs.io/ のドキュメントは比較的アクセスしやすく、完全です。

仕様
==========================================================================================

.. toctree::
   :maxdepth: 2

   spec/index

索引とテーブル
==========================================================================================

* :ref:`genindex`
* :ref:`search`

.. _contact:

ディスカッションとサポート
==========================================================================================

* `ユーザーヘルプフォーラム <https://github.com/python/typing/discussions>`_
* `Gitter でのユーザーチャット <http://gitter.im/python/typing>`_
* `開発者フォーラム <https://discuss.python.org/c/typing/32>`_
* `開発者メーリングリスト（アーカイブ） <https://mail.python.org/archives/list/typing-sig@python.org/>`_

型関連ツール
==========================================================================================

型チェッカー
------------------------------------------------------------------------------------------

* `mypy <http://mypy-lang.org/>`_, 型チェッカーのリファレンス実装。
* `pyre <https://pyre-check.org/>`_, OCaml で書かれ、パフォーマンスに最適化された型チェッカー。
* `pyright <https://github.com/microsoft/pyright>`_, 速度を重視した型チェッカー。
* `pytype <https://google.github.io/pytype/>`_, 型注釈のないコードの型をチェックおよび推論する型チェッカー。

開発環境
------------------------------------------------------------------------------------------

* `PyCharm <https://www.jetbrains.com/pycharm/>`_, 型スタブを型チェックとコード補完の両方にサポートする IDE。
* `Visual Studio Code <https://code.visualstudio.com/>`_, mypy、pyright、または
  `Pylance <https://marketplace.visualstudio.com/items?itemName=ms-python.vscode-pylance>`_
  拡張機能を使用して型チェックをサポートするコードエディタ。

リンターとフォーマッター
------------------------------------------------------------------------------------------

* `black <https://black.readthedocs.io/>`_, 型スタブファイルをサポートするコードフォーマッター。
* `flake8-pyi <https://github.com/ambv/flake8-pyi>`_, 型スタブをサポートする
  `flake8 <https://flake8.pycqa.org/>`_ リンターのプラグイン。
* `ruff <https://astral.sh/ruff>`_, ほとんどの ``flake8-pyi`` ルールをサポートする速度重視のリンター。

型ヒントとスタブの統合
------------------------------------------------------------------------------------------

* `autotyping <https://github.com/JelleZijlstra/autotyping>`_, コンテキストから単純な型を推論し、それらをインライン型ヒントとして挿入するツール。
* `merge-pyi
  <https://google.github.io/pytype/developers/tools.html#merge_pyi>`_,
  .pyi シグネチャを Python ソースコードのインライン型ヒントとして統合する
  `libCST <https://libcst.readthedocs.io/en/latest/>`_ の ``ApplyTypeAnnotationsVisitor`` の薄いラッパー。

型に関する PEP
==========================================================================================

すべての型関連の PEP のリストについては、https://peps.python.org/topic/typing を参照してください。
