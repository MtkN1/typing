.. _writing_stubs:

******************************************************************************************
スタブファイルの作成と維持
******************************************************************************************

スタブファイルは、Python モジュールに型情報を提供する手段です。
完全なリファレンスについては、:ref:`stub-files` を参照してください。

スタブの維持は、実装から分離されているため、少し面倒です。 このページでは、スタブの作成と維持を少しでも楽にするためのツールと、スタブの内容とスタイルに関するベストプラクティスを紹介します。

スタブを生成するためのツール
==========================================================================================

stubgen
-------

stubgen は、`mypy <https://github.com/python/mypy>`__ にバンドルされているツールで、基本的なスタブを生成するために使用できます。 これらのスタブは基本的な出発点として機能し、ほとんどの型はデフォルトで ``Any`` になります。

.. code-block:: console

    stubgen -p my_great_package

詳細については、`stubgen docs <https://mypy.readthedocs.io/en/stable/stubgen.html>`__ を参照してください。

pyright
-------

pyright には基本的なスタブを生成するツールが含まれています。 stubgen と同様に、これらの生成されたスタブは出発点として機能します。

.. code-block:: console

    pyright --createstub my_great_package

詳細については、`pyright docs <https://github.com/microsoft/pyright/blob/main/docs/type-stubs.md#generating-type-stubs-from-command-line>`__ を参照してください。

monkeytype
----------

monkeytype は少し異なるアプローチを取ります。コードを実行し (おそらくテストを通じて)、monkeytype は実行時に観察された型を収集してスタブを生成します。

.. code-block:: console

    monkeytype run script.py
    monkeytype stub my_great_package

詳細については、`monkeytype docs <https://monkeytype.readthedocs.io/en/latest/>`__ を参照してください。

スタブを維持するためのツール
==========================================================================================

stubtest
--------

stubtest は、`mypy <https://github.com/python/mypy>`__ にバンドルされているツールです。

stubtest は、スタブファイルと実装の間の不一致を見つけます。 これは、スタブ定義をインポートしてランタイムイントロスペクション (``inspect`` モジュールを介して) を使用して見つけたものと比較することで行います。

.. code-block:: console

    stubtest my_great_package

詳細については、`stubtest docs <https://mypy.readthedocs.io/en/stable/stubtest.html>`__ を参照してください。

flake8-pyi
------------------------------------------------------------------------------------------

flake8-pyi は、スタブファイルの一般的な問題をリントする `flake8 <https://flake8.pycqa.org/en/latest/>`__ プラグインです。

.. code-block:: console

    flake8 my_great_package

詳細については、`flake8-pyi docs <https://github.com/PyCQA/flake8-pyi>`__ を参照してください。

スタブの使用に関する型チェックの実行
------------------------------------------------------------------------------------------

スタブに対して型チェッカーを実行するだけで、欠落している注釈の検出などの単純な問題から、リスコフの置換可能性の確認や問題のあるオーバーロードの検出などの複雑な問題まで、いくつかの問題を検出できます。

`typeshed <https://github.com/python/typeshed/>`__ の `setup for testing stubs <https://github.com/python/typeshed/blob/main/tests/README.md>`__ を調べると参考になるかもしれません。

..
   TODO: 特定の型チェッカーの例や設定を追加することを検討してください

パッケージの使用に関する型チェックの実行
------------------------------------------------------------------------------------------

パッケージを使用するコードベースにアクセスできる場合 (おそらくパッケージのテスト)、それに対して型チェッカーを実行することで、特に偽陽性の問題を検出するのに役立ちます。

パッケージに特に複雑な側面がある場合は、難しい定義のための専用の型テストを作成することも検討できます。 詳細については、:ref:`testing` を参照してください。

スタブの内容
==========================================================================================

このセクションでは、スタブファイルに含める要素や除外する要素に関するベストプラクティスを文書化しています。

スタブから除外されるモジュール
------------------------------------------------------------------------------------------

すべてのモジュールがスタブに含まれるべきではありません。

次のものを除外することをお勧めします:

1. 実装の詳細。顕著な例として `multiprocessing/popen_spawn_win32.py <https://github.com/python/cpython/blob/main/Lib/multiprocessing/popen_spawn_win32.py>`_
2. インポートされることを意図していないモジュール (例: ``__main__.py``)
3. 単一の ``_`` 文字で始まる保護されたモジュール。ただし、必要に応じて保護されたモジュールを追加することはできます (以下の :ref:`undocumented-objects` セクションを参照してください)
4. テスト

パブリックインターフェース
------------------------------------------------------------------------------------------

スタブには、カバーするモジュールの完全なパブリックインターフェース (クラス、関数、定数など) を含める必要がありますが、インターフェースの一部が何であるかは必ずしも明確ではありません。

次のものは常に含める必要があります:

* モジュールのドキュメントに記載されているすべてのオブジェクト。
* ``__all__`` に含まれるすべてのオブジェクト (存在する場合)。

他のオブジェクトは、アンダースコアで始まっていない場合や、実際に使用されている場合に含めることができます。 (次のセクションを参照してください。)

.. _undocumented-objects:

未文書のオブジェクト
------------------------------------------------------------------------------------------

未文書のオブジェクトは、``# undocumented`` という形式のコメントでマークされている限り、含めることができます。

例::

    def list2cmdline(seq: Sequence[str]) -> str: ...  # undocumented

このような未文書のオブジェクトは、オブジェクトを省略するとユーザーが混乱する可能性があるため許可されています。 ユーザーが「モジュール X に属性 Y がありません」というエラーを見た場合、そのエラーがコードにバグがあるために発生したのか、スタブが間違っているために発生したのかを知ることはできません。 プライベートオブジェクトの使用を型チェッカーが指摘することも役立ちますが、誤検知 (正しいコードに対する型エラー) よりも偽陰性 (誤ったコードに対するエラーなし) の方が望ましいです。 さらに、プライベートオブジェクトであっても、型チェッカーは不正な型が使用されたことを指摘するのに役立ちます。

``__all__``
------------------------------------------------------------------------------------------

スタブファイルには、ランタイムでも存在する場合にのみ ``__all__`` 変数を含める必要があります。 その場合、``__all__`` の内容はスタブとランタイムで同一である必要があります。 ランタイムが動的に要素を追加または削除する場合 (たとえば、特定の関数が特定のシステム構成でのみ利用可能な場合)、スタブにはすべての可能な要素を含めます。

スタブ専用のオブジェクト
------------------------------------------------------------------------------------------

ランタイムに存在しない定義は、型を表現するのに役立つため、スタブに含めることができます。 ユーザーに意図的に公開されていない限り (以下を参照)、そのような定義は名前の前にアンダースコアを付けてプライベートとしてマークする必要があります。

はい::

    _T = TypeVar("_T")
    _DictList: TypeAlias = dict[str, list[int | None]]

いいえ::

    T = TypeVar("T")
    DictList: TypeAlias = dict[str, list[int | None]]

場合によっては、スタブ専用のクラスをスタブのユーザーに利用可能にすることが望ましい場合があります。 たとえば、ライブラリが使用可能なランタイム型を提供しないパブリックメソッドの戻り値を型指定するためです。 そのようなオブジェクトをマークするには、``typing.type_check_only`` デコレータを使用します::

  from typing import Protocol, type_check_only

  @type_check_only
  class Readable(Protocol):
      def read(self) -> str: ...

  def get_reader() -> Readable: ...

構造型
------------------------------------------------------------------------------------------

前のセクションの ``Readable`` の例で見たように、スタブ専用のオブジェクトの一般的な使用法は、その構造によって最もよく説明される型をモデル化することです。 これらのオブジェクトはプロトコル (:pep:`544`) と呼ばれ、単純な構造型を記述するために自由に使用することが奨励されています。

不完全なスタブ
------------------------------------------------------------------------------------------

部分的なスタブは特に大規模なパッケージにとって有用ですが、次のガイドラインに従う必要があります:

* 含まれる関数とメソッドはすべての引数をリストする必要がありますが、引数は注釈なしにすることができます。
* 注釈なしまたは部分的に注釈された値をマークするために ``Any`` を使用しないでください。 関数のパラメータと戻り値は注釈なしにしてください。 その他のすべての場合、``_typeshed.Incomplete`` を使用します
  (`documentation <https://github.com/python/typeshed/blob/main/stdlib/_typeshed/README.md>`_)::

    from _typeshed import Incomplete

    field1: Incomplete
    field2: dict[str, Incomplete]

    def foo(x): ...

* 部分的なクラスには、``_typeshed.Incomplete`` でマークされた ``__getattr__()`` メソッドを含める必要があります (以下の例を参照)。
* 部分的なモジュール (クラス、関数、属性の一部またはすべてが欠けているモジュール) には、``_typeshed.Incomplete`` でマークされたトップレベルの ``__getattr__()`` 関数を含める必要があります (以下の例を参照)。
* 部分的なパッケージ (1 つ以上のサブモジュールが欠けているパッケージ) には、不完全としてマークされた ``__init__.pyi`` スタブが必要です (上記を参照)。 より良い代替案は、すべてのサブモジュールの空のスタブを作成し、それぞれを個別に不完全としてマークすることです。

部分的なクラス ``Foo`` と部分的に注釈された関数 ``bar()`` を含む部分的なモジュールの例::

    from _typeshed import Incomplete

    def __getattr__(name: str) -> Incomplete: ...

    class Foo:
        def __getattr__(self, name: str) -> Incomplete: ...
        x: int
        y: str

    def bar(x: str, y, *, z=...): ...

属性アクセス
------------------------------------------------------------------------------------------

Python には、属性アクセスをカスタマイズするためのいくつかのメソッドがあります: ``__getattr__``, ``__getattribute__``, ``__setattr__``, および ``__delattr__``。 これらのうち、``__getattr__`` と ``__setattr__`` はスタブに含める必要がある場合があります。

不完全な定義をマークすることに加えて、クラスまたはモジュールが任意の名前にアクセスできる場合は、``__getattr__`` を含める必要があります。 たとえば、次のクラスを考えてみましょう::

  class Foo:
      def __getattribute__(self, name):
          return self.__dict__.setdefault(name)

適切なスタブ定義は次のとおりです::

  from typing import Any

  class Foo:
      def __getattr__(self, name: str) -> Any | None: ...

スタブでサポートされることが保証されているのは ``__getattr__`` のみであり、``__getattribute__`` ではないことに注意してください。

一方、次のクラスを考えてみましょう::

  class ComplexNumber:
      def __init__(self, n):
          self._n = n
      def __getattr__(self, name):
          if name in ("real", "imag"):
              return getattr(self._n, name)
          raise AttributeError(name)

この場合、スタブは属性を個別にリストする必要があります::

  class ComplexNumber:
      @property
      def real(self) -> float: ...
      @property
      def imag(self) -> float: ...
      def __init__(self, n: complex) -> None: ...

クラスが任意の名前を設定でき、型を制限する場合は、``__setattr__`` を含める必要があります。 たとえば::

  class IntHolder:
      def __setattr__(self, name, value):
          if isinstance(value, int):
              return super().__setattr__(name, value)
          raise ValueError(value)

適切なスタブ定義は次のとおりです::

  class IntHolder:
      def __setattr__(self, name: str, value: int) -> None: ...

``__delattr__`` はスタブに含めるべきではありません。

最後に、``__getattr__`` と ``__setattr__`` が存在する場合でも、既知の属性を個別に定義することをお勧めします。

定数
------------------------------------------------------------------------------------------

定数の値が重要な場合は、それを ``Final`` としてマークし、その値に割り当てます。

はい::

    TEL_LANDLINE: Final = "landline"
    TEL_MOBILE: Final = "mobile"
    DAY_FLAG: Final = 0x01
    NIGHT_FLAG: Final = 0x02

いいえ::

    TEL_LANDLINE: str
    TEL_MOBILE: str
    DAY_FLAG: int
    NIGHT_FLAG: int

オーバーロード
------------------------------------------------------------------------------------------

オーバーロードされた関数とメソッドのすべてのバリアントには、``@overload`` デコレータが必要です。 実装の最終的な非 ``@overload`` デコレータ付きの定義を含めないでください。

はい::

  @overload
  def foo(x: str) -> str: ...
  @overload
  def foo(x: float) -> int: ...

いいえ::

  @overload
  def foo(x: str) -> str: ...
  @overload
  def foo(x: float) -> int: ...
  def foo(x: str | float) -> Any: ...

デコレータ
------------------------------------------------------------------------------------------

すべての主要な型チェッカーによって理解される効果を持つ、ここにリストされているデコレータ (:ref:`stub-decorators`) のみを含めます。 他のデコレータの動作は、型に組み込む必要があります。 たとえば、次の関数の場合::

  import contextlib
  @contextlib.contextmanager
  def f():
      yield 42

スタブ定義は次のようになります::

  from contextlib import AbstractContextManager
  def f() -> AbstractContextManager[int]: ...

ドキュメントまたは実装
------------------------------------------------------------------------------------------

ライブラリの文書化された型がコード内の実際の型と異なる場合があります。 そのような場合、スタブの作成者は最良の判断を使用する必要があります。 次の 2 つの例を考えてみましょう::

  def print_elements(x):
      """リスト x のすべての要素を印刷します。"""
      for y in x:
          print(y)

  def maybe_raise(x):
      """x (ブール値) が true の場合にエラーを発生させます。"""
      if x:
          raise ValueError()

``print_elements`` の実装は、文書化された ``list`` 型にもかかわらず、任意の反復可能オブジェクトを受け取ります。 この場合、引数を ``Iterable[object]`` として注釈し、引数に対して抽象型を優先するという :ref:`best practice<argument-return-practices>` に従います。

一方、``maybe_raise`` については、実装が任意のオブジェクトを受け入れるにもかかわらず、引数を ``bool`` として注釈する方が良いです。 これにより、意図せずに ``None`` を渡すなどの一般的な間違いを防ぐことができます。

疑問がある場合は、ライブラリのメンテナーに意図を尋ねることを検討してください。

スタイルガイド
==========================================================================================

このセクションの推奨事項は、スタブに一貫したスタイルを提供したいスタブ作成者を対象としています。 型チェッカーは、これらの推奨事項に従わないスタブを拒否するべきではありませんが、リンターはそれらについて警告することがあります。

スタブファイルは一般的に、Python コードのスタイルガイド (:pep:`8`) と :ref:`best-practices` に従う必要があります。 スタブファイルの異なる構造を考慮し、より簡潔なファイルを作成することを目的としたいくつかの例外があります。

最大行長
------------------------------------------------------------------------------------------

スタブファイルは、1 行あたり 130 文字に制限する必要があります。

空行
------------------------------------------------------------------------------------------

関数、メソッド、フィールドの間に空行を使用しないでください。ただし、1 つの空行でグループ化するために使用することはできます。 本文が空でないクラスの周りには 1 つの空行を使用します。 本文が空のクラスの間に空行を使用しないでください。ただし、グループ化のために使用することはできます。

はい::

    def time_func() -> None: ...
    def date_func() -> None: ...

    def ip_func() -> None: ...

    class Foo:
        x: int
        y: int
        def __init__(self) -> None: ...

    class MyError(Exception): ...
    class AnotherError(Exception): ...

いいえ::

    def time_func() -> None: ...

    def date_func() -> None: ...  # 不要な空行を残さないでください

    def ip_func() -> None: ...


    class Foo:  # 上に 1 つの空行のみを残します
        x: int
    class MyError(Exception): ...  # クラスの間に空行を残します

モジュールレベルの属性
------------------------------------------------------------------------------------------

モジュールレベルの属性に対して不要な代入を使用しないでください。

はい::

    CONST: Literal["const"]
    x: int
    y: Final = 0  # この代入は追加の型情報を伝えます

いいえ::

    CONST = "const"
    x: int = 0
    y: float = ...
    z = 0  # type: int
    a = ...  # type: int

.. _stub-style-classes:

クラス
------------------------------------------------------------------------------------------

本文のないクラスは、クラス定義と同じ行に省略記号リテラル ``...`` を使用する必要があります。

はい::

    class MyError(Exception): ...

いいえ::

    class MyError(Exception):
        ...
    class AnotherError(Exception): pass

インスタンス属性とクラス変数は、モジュールレベルの属性と同じ推奨事項に従います:

はい::

    class Foo:
        c: ClassVar[str]
        x: int

    class Color(Enum):
        # 代入に型注釈がない場合は、列挙メンバーを示すための慣例です。
        RED = 1

いいえ::

    class Foo:
        c: ClassVar[str] = ""
        d: ClassVar[int] = ...
        x = 4
        y: int = ...

関数とメソッド
------------------------------------------------------------------------------------------

キーワード専用および位置またはキーワード引数については、実装と同じ引数名を使用してください。そうしないと、キーワード引数の使用が失敗します。

デフォルト値については、「単純な」デフォルト値 (``None``, ブール値, 整数, バイト, 文字列, および浮動小数点数) のリテラル値を使用します。 より複雑なデフォルト値の代わりに省略記号リテラル ``...`` を使用します。 デフォルトが ``None`` の場合は、明示的な ``X | None`` 注釈を使用します。

はい::

    def foo(x: int = 0) -> None: ...
    def bar(y: str | None = None) -> None: ...

いいえ::

    def foo(x: X = X()) -> None: ...
    def bar(y: str = None) -> None: ...

メソッド定義で ``self`` および ``cls`` を注釈しないでください。ただし、型変数を参照する場合は除きます。

はい::

    _T = TypeVar("_T")

    class Foo:
        def bar(self) -> None: ...
        @classmethod
        def create(cls: type[_T]) -> _T: ...

いいえ::

    class Foo:
        def bar(self: Foo) -> None: ...
        @classmethod
        def baz(cls: type[Foo]) -> int: ...

関数とメソッドの本文は、閉じ括弧とコロンと同じ行に省略記号リテラル ``...`` のみで構成する必要があります。

はい::

    def to_int1(x: str) -> int: ...
    def to_int2(
        x: str,
    ) -> int: ...

いいえ::

    def to_int1(x: str) -> int:
        return int(x)
    def to_int2(x: str) -> int:
        ...
    def to_int3(x: str) -> int: pass

言語機能
------------------------------------------------------------------------------------------

スタブが古い Python バージョンを対象としている場合でも、利用可能な最新の言語機能を使用します。 たとえば、Python 3.7 では ``async`` キーワードが追加されました (:pep:`492` を参照)。 スタブはコルーチンをマークするためにそれを使用する必要があります。 一方、Python 3.12 で導入された :pep:`695` の ``type`` ソフトキーワードは、Python 3.11 が 2027 年 10 月にサポート終了するまでスタブで使用するべきではありません。

順方向参照の周りに引用符を使用せず、``__future__`` インポートを使用しないでください。 詳細については、:ref:`stub-file-syntax` を参照してください。

はい::

    class Py35Class:
        x: int
        forward_reference: OtherClass

    class OtherClass: ...

いいえ::

    class Py35Class:
        x = 0  # type: int
        forward_reference: 'OtherClass'

    class OtherClass: ...

NamedTuple と TypedDict
------------------------------------------------------------------------------------------

``typing.NamedTuple`` および ``typing.TypedDict`` には、スタイルガイドの :ref:`stub-style-classes` セクションに従ってクラスベースの構文を使用します。

はい::

    from typing import NamedTuple, TypedDict

    class Point(NamedTuple):
        x: float
        y: float

    class Thing(TypedDict):
        stuff: str
        index: int

いいえ::

    from typing import NamedTuple, TypedDict
    Point = NamedTuple("Point", [('x', float), ('y', float)])
    Thing = TypedDict("Thing", {'stuff': str, 'index': int})

組み込みジェネリクス
------------------------------------------------------------------------------------------

:pep:`585` 組み込みジェネリクスがサポートされており、``typing`` から対応する型の代わりに使用する必要があります::

    from collections import defaultdict

    def foo(t: type[MyClass]) -> list[int]: ...
    x: defaultdict[int]

``typing`` の代わりに ``collections.abc`` からのインポートを使用することは一般的に可能であり、推奨されます::

    from collections.abc import Iterable

    def foo(iter: Iterable[int]) -> None: ...

ユニオン
------------------------------------------------------------------------------------------

短縮形の `|` 構文を使用してユニオンを宣言することが推奨されており、すべての型チェッカーでサポートされています::

  def foo(x: int | str) -> int | None: ...  # 推奨
  def foo(x: Union[int, str]) -> Optional[int]: ...  # ok
