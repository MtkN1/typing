.. _testing:

******************************************************************************************
型注釈の品質のテストと保証
******************************************************************************************

注釈の精度のテスト
==========================================================================================

型注釈を含むパッケージを作成する場合、著者は公開する注釈が期待に沿っていることを検証したいと考えるかもしれません。
これは、公開された注釈がパッケージのパブリック インターフェイスの一部であるライブラリの作成者にとって特に重要です。

この問題にはいくつかのアプローチがあり、このドキュメントではそのいくつかを紹介します。

.. note::

    簡単のために、型チェックは ``mypy`` で行われると仮定します。
    これらの戦略の多くは、他の型チェッカーにも適用できます。

``assert_type`` と ``--warn-unused-ignores`` を使用したテスト
------------------------------------------------------------------------------------------

通常の Python ファイルを作成し、``typing_tests/`` などの専用ディレクトリに配置して、型注釈の特定のプロパティをアサートするというアイデアです。

``assert_type`` (``mypy`` 0.950 以上) を使用すると、型注釈が予期される型を生成することを確認できます。

次のファイルがテスト対象である場合:

.. code-block:: python

    # foo.py
    def bar(x: int) -> str:
        return str(x)

次のファイルは ``foo.py`` をテストします:

.. code-block:: python

    from typing_extensions import assert_type

    assert_type(bar(42), str)

``mypy --warn-unused-ignores`` を巧妙に使用すると、特定の式が型指定されているかどうかを確認できます。
有効な式と ``type: ignore`` コメントで注釈された無効な式を一緒に配置するというアイデアです。
これらのファイルで ``mypy --warn-unused-ignores`` を実行すると、パスするはずです。

この戦略は、テスト対象の型に関する強力な保証を提供しませんが、追加のツールは必要ありません。

次のファイルがテスト対象である場合:

.. code-block:: python

    # foo.py
    def bar(x: int) -> str:
        return str(x)

次のファイルは ``foo.py`` をテストします:

.. code-block:: python

    bar(42)
    bar("42")  # type: ignore [arg-type]
    bar(y=42)  # type: ignore [call-arg]
    r1: str = bar(42)
    r2: int = bar(42)  # type: ignore [assignment]

``mypy.api.run`` からの ``reveal_type`` 出力のチェック
------------------------------------------------------------------------------------------

``mypy`` は、Python プロセスから ``mypy`` を呼び出すための ``api`` というサブパッケージを提供します。
``reveal_type`` と組み合わせて、式から ``reveal_type`` 出力を取得する関数を記述するために使用できます。
それが得られたら、テストはそれに対して文字列と正規表現の一致をアサートできます。

このアプローチでは、優れたテスト エクスペリエンスを提供するためのヘルパー セットを記述する必要があり、テスト ケースごとに mypy を 1 回実行します (これには時間がかかる場合があります)。
ただし、``mypy`` と選択したテスト フレームワークのみに基づいて構築されます。

次の例は、任意のフレームワークで記述されたテスト スイートに統合できます:

.. code-block:: python

    import re
    from mypy import api

    def get_reveal_type_output(filename):
        result = api.run([filename])
        stdout = result[0]
        match = re.search(r'note: Revealed type is "([^"]+)"', stdout)
        assert match is not None
        return match.group(1)


たとえば、上記を使用して一時ファイルを生成し、それを ``get_reveal_type_output`` への入力として使用する ``run_reveal_type`` pytest フィクスチャを提供できます:

.. code-block:: python

    import os
    import pytest

    @pytest.fixture
    def _in_tmp_path(tmp_path):
        cur = os.getcwd()
        try:
            os.chdir(tmp_path)
            yield
        finally:
            os.chdir(cur)

    @pytest.fixture
    def run_reveal_type(tmp_path, _in_tmp_path):
        content_path = tmp_path / "reveal_type_test.py"

        def func(code_snippet, *, preamble = ""):
            content_path.write_text(preamble + f"reveal_type({code_snippet})")
            return get_reveal_type_output("reveal_type_test.py")

        return func


詳細については、`mypy.api に関するドキュメント <https://mypy.readthedocs.io/en/stable/extending_mypy.html#integrating-mypy-into-another-python-application>`_ を参照してください。

pytest-mypy-plugins
------------------------------------------------------------------------------------------

`pytest-mypy-plugins <https://github.com/typeddjango/pytest-mypy-plugins>`_ は、型テスト ケースを YAML データとして定義する ``pytest`` 用のプラグインです。
テスト ケースは ``mypy`` を通じて実行され、``reveal_type`` の出力をアサートできます。

このプロジェクトは、``pytest`` パラメータ化テストやテストごとの ``mypy`` 構成など、複雑な型の配置をサポートしています。
テストを実行するには ``pytest`` を使用する必要があり、テスト ケースごとにサブプロセスで ``mypy`` を実行します。

これは、``pytest-mypy-plugins`` を使用したパラメータ化テストの例です:

.. code-block:: yaml

    - case: with_params
      parametrized:
        - val: 1
          rt: builtins.int
        - val: 1.0
          rt: builtins.float
      main: |
        reveal_type({[ val }})  # N: Revealed type is '{{ rt }}'

型の完全性の向上
==========================================================================================

多くのライブラリの目標の 1 つは、「完全に型注釈されている」ことを確認することです。つまり、すべての関数、クラス、およびオブジェクトに完全かつ正確な型注釈を提供することです。
完全な注釈を持つことは「型の完全性」または「型カバレッジ」と呼ばれます。

ライブラリの型の完全性スコアを向上させるためのヒントをいくつか紹介します。

-  型の完全性をテスト プロセスの出力にします。 いくつかの型チェッカーには、有用な出力、警告、さらにはレポートを生成するためのオプションがあります。
-  パッケージにテストやサンプル コードが含まれている場合は、それらを配布から削除することを検討してください。 含める正当な理由がある場合は、アンダースコアで始まるディレクトリに配置して、ライブラリのインターフェイスの一部と見なされないようにすることを検討してください。
-  実装の詳細を意図したサブモジュールがパッケージに含まれている場合は、それらのファイルの名前をアンダースコアで始まるように変更します。
-  シンボルがライブラリのインターフェイスの一部として意図されておらず、実装の詳細と見なされる場合は、アンダースコアで始まるように名前を変更します。 その後、プライベートと見なされ、型の完全性チェックから除外されます。
-  パッケージが他のライブラリの型を公開している場合は、これらの他のライブラリのメンテナーと協力して型の完全性を達成します。

.. warning::

    さまざまな型チェッカーが型カバレッジを評価し、より良い型カバレッジを達成するのに役立つ方法は異なる場合があります。 上記の推奨事項の一部は、使用する型チェック ツールによっては役に立たない場合があります。

``mypy`` 不許可オプション
------------------------------------------------------------------------------------------

``mypy`` には、型指定されていないコードを検出できるいくつかのオプションがあります。
詳細については、`これらのオプションに関する mypy ドキュメント <https://mypy.readthedocs.io/en/latest/command_line.html#untyped-definitions-and-calls>`_ を参照してください。

型指定されていないデータに対して ``mypy`` エラーを発生させる基本的な使用法は次のとおりです::

    mypy --disallow-untyped-defs
    mypy --disallow-incomplete-defs

``pyright`` 型検証
------------------------------------------------------------------------------------------

pyright には、型の完全性を検証するための特別なコマンド ライン フラグ ``--verifytypes`` があります。
詳細については、`型の完全性の検証に関する pyright ドキュメント <https://github.com/microsoft/pyright/blob/main/docs/typed-libraries.md#verifying-type-completeness>`_ を参照してください。

``mypy`` レポート
------------------------------------------------------------------------------------------

``mypy`` には、分析に関するレポートを生成するためのいくつかのオプションがあります。
詳細については、`レポート生成に関する mypy ドキュメント <https://mypy.readthedocs.io/en/stable/command_line.html#report-generation>`_ を参照してください。
