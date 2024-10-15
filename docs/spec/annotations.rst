.. _`type-annotations`:

型注釈
==========================================================================================

注釈の意味
------------------------------------------------------------------------------------------

型システムは、以下のセクションで説明するいくつかの拡張を持つ :pep:`3107` スタイルの注釈を活用します。 基本的な形式では、型ヒントはクラスで関数注釈スロットを埋めることによって使用されます::

  def greeting(name: str) -> str:
      return 'Hello ' + name

これは、``name`` 引数の期待される型が ``str`` であることを示しています。 同様に、期待される戻り値の型は ``str`` です。

特定の引数型に :term:`assignable` な式もその引数に対して受け入れられます。 同様に、注釈付きの戻り値の型に代入可能な式は、関数から返すことができます。

.. _`missing-annotations`:

注釈のない関数はすべて、すべての引数と戻り値の型に :ref:`Any` 注釈があるものとして扱うことができます。 型チェッカーは、欠落している注釈に対してより正確な型を推測することもオプションで行うことができます。

型チェッカーは、注釈のない関数の本体を完全に無視する（型チェックしない）こともできますが、この動作は必須ではありません。

すべての引数と戻り値に注釈を付けることが推奨されますが、必須ではありません。 チェックされた関数の場合、引数と戻り値のデフォルトの注釈は ``Any`` です。 例外は、インスタンスメソッドとクラスメソッドの最初の引数です。 それが注釈されていない場合、それはインスタンスメソッドの場合は含まれているクラスの型を持つと見なされ、クラスメソッドの場合は含まれているクラスオブジェクトに対応する型オブジェクト型を持つと見なされます。 たとえば、クラス ``A`` では、インスタンスメソッドの最初の引数は暗黙的に ``A`` 型を持ちます。 クラスメソッドでは、最初の引数の正確な型は利用可能な型表記を使用して表すことはできません。

（注：``__init__`` の戻り値の型は ``-> None`` で注釈を付ける必要があります。 これは微妙な理由です。 ``__init__`` が ``-> None`` の戻り注釈を持つと仮定すると、引数なしの注釈なしの ``__init__`` メソッドは依然として型チェックされるべきでしょうか？ これを曖昧にしたり、例外に対する例外を導入する代わりに、``__init__`` に戻り注釈を付ける必要があると単に言います。 デフォルトの動作は他のメソッドと同じです。）

型チェッカーは、与えられた注釈と一致するかどうかを確認するために、チェックされた関数の本体をチェックすることが期待されます。 注釈は、他のチェックされた関数に現れる呼び出しの正確性を確認するためにも使用される場合があります。

型チェッカーは、必要な情報をできるだけ多く推測しようとすることが期待されます。 最低限の要件は、組み込みのデコレータ ``@property``、``@staticmethod``、および ``@classmethod`` を処理することです。

.. _valid-types:

型と注釈の式
------------------------------------------------------------------------------------------

*型式* および *注釈式* という用語は、型システムで使用される特定のサブセットの Python 式を指します。 すべての型式は注釈式でもありますが、すべての注釈式が型式であるわけではありません。

.. _`type-expression`:

*型式* は、型を正しく表現する任意の式です。 型式は常に注釈式として受け入れられ、さまざまな他の場所でも使用されます。 具体的には、型式は次の場所で使用されます。

* 型注釈内（常に注釈式の一部として）
* :ref:`cast() <cast>` の最初の引数
* :ref:`assert_type() <assert-type>` の2番目の引数
* ``TypeVar`` の境界および制約（古い構文または Python 3.12 のネイティブ構文のいずれかを使用して作成された場合）
* 型エイリアスの定義（``type`` ステートメント、古い代入構文、または ``TypeAliasType`` コンストラクタを使用して作成された場合）
* ジェネリッククラスの型引数（ベースクラスまたはコンストラクタ呼び出しに現れる場合）
* :ref:`TypedDict <typeddict>` および :ref:`NamedTuple <namedtuple>` 型を作成するための機能形式のフィールドの定義
* :ref:`NewType <newtype>` の定義

.. _`annotation-expression`:

*注釈式* は、注釈コンテキスト（関数パラメータ注釈、関数戻り注釈、または変数注釈）で使用することが許可される式です。 一般に、注釈式は型式であり、1つ以上の :term:`type qualifiers <type qualifier>` または `Annotated` で囲まれている場合があります。 各型修飾子は特定のコンテキストでのみ有効です。 注釈式は型システムで型注釈として有効な唯一の式ですが、Python 言語自体にはそのような制限はありません。 任意の式が許可されます。

注釈は、関数が定義された時点で例外を発生させずに評価される有効な式でなければなりません（ただし、:ref:`forward-references` を参照してください）。

.. _`expression-grammar`:

次の文法は、型式および注釈式で許可される要素を説明しています。

.. productionlist:: expression-grammar
    annotation_expression: <Required> '[' `annotation_expression` ']'
                         : | <NotRequired> '[' `annotation_expression` ']'
                         : | <ReadOnly> '[' `annotation_expression`']'
                         : | <ClassVar> '[' `annotation_expression`']'
                         : | <Final> '[' `annotation_expression`']'
                         : | <InitVar> '[' `annotation_expression` ']'
                         : | <Annotated> '[' `annotation_expression` ','
                         :               expression (',' expression)* ']'
                         : | <TypeAlias>
                         :       (valid only in variable annotations)
                         : | `unpacked`
                         :       (valid only for *args annotations)
                         : | <Unpack> '[' name ']'
                         :       (where name refers to an in-scope TypedDict;
                         :        valid only in **kwargs annotations)
                         : | `string_annotation`
                         :       (must evaluate to a valid `annotation_expression`)
                         : | name '.' 'args'
                         :      (where name must be an in-scope ParamSpec;
                         :       valid only in *args annotations)
                         : | name '.' 'kwargs'
                         :       (where name must be an in-scope ParamSpec;
                         :        valid only in **kwargs annotations)
                         : | `type_expression`
    type_expression: <Any>
                   : | <Self>
                   :       (valid only in some contexts)
                   : | <LiteralString>
                   : | <NoReturn>
                   : | <Never>
                   : | <None>
                   : | name
                   :       (where name must refer to a valid in-scope class,
                   :        type alias, or TypeVar)
                   : | name '[' (`maybe_unpacked` | `type_expression_list`)
                   :        (',' (`maybe_unpacked` | `type_expression_list`))* ']'
                   :       (the `type_expression_list` form is valid only when
                   :        specializing a ParamSpec)
                   : | name '[' '(' ')' ']'
                   :       (denoting specialization with an empty TypeVarTuple)
                   : | <Literal> '[' expression (',' expression) ']'
                   :       (see documentation for Literal for restrictions)
                   : | `type_expression` '|' `type_expression`
                   : | <Optional> '[' `type_expression` ']'
                   : | <Union> '[' `type_expression` (',' `type_expression`)* ']'
                   : | <type> '[' <Any> ']'
                   : | <type> '[' name ']'
                   :       (where name must refer to a valid in-scope class
                   :        or TypeVar)
                   : | <Callable> '[' '...' ',' `type_expression` ']'
                   : | <Callable> '[' name ',' `type_expression` ']'
                   :       (where name must be a valid in-scope ParamSpec)
                   : | <Callable> '[' <Concatenate> '[' (`type_expression` ',')+
                   :              (name | '...') ']' ',' `type_expression` ']'
                   :       (where name must be a valid in-scope ParamSpec)
                   : | <Callable> '[' '[' `maybe_unpacked` (',' `maybe_unpacked`)*
                   :              ']' ',' `type_expression` ']'
                   : | `tuple_type_expression`
                   : | <Annotated> '[' `type_expression` ','
                   :               expression (',' expression)* ']'
                   : | <TypeGuard> '[' `type_expression` ']'
                   :       (valid only in some contexts)
                   : | <TypeIs> '[' `type_expression` ']'
                   :       (valid only in some contexts)
                   : | `string_annotation`
                   :       (must evaluate to a valid `type_expression`)
    maybe_unpacked: `type_expression` | `unpacked`
    unpacked: '*' `unpackable`
            : | <Unpack> '[' `unpackable` ']'
    unpackable: `tuple_type_expression``
              : | name
              :       (where name must refer to an in-scope TypeVarTuple)
    tuple_type_expression: <tuple> '[' '(' ')' ']'
                         :      (representing an empty tuple)
                         : | <tuple> '[' `type_expression` ',' '...' ']'
                         :       (representing an arbitrary-length tuple)
                         : | <tuple> '[' `maybe_unpacked` (',' `maybe_unpacked`)* ']'
    string_annotation: string
                     :     (must be a string literal that is parsable
                     :      as Python code; see "String annotations")
    type_expression_list: '[' `type_expression` (',' `type_expression`)* ']'
                        : | '[' ']'

注：

* 文法は、コードがすでに Python コードとして解析されていることを前提としており、AST の構造に緩やかに従います。 コメントや空白などの構文的な詳細は無視されます。

* ``<Name>`` は :term:`special form` を指します。 ほとんどの特殊形式は :py:mod:`typing` または ``typing_extensions`` からインポートする必要がありますが、``None``、``InitVar``、``type``、および ``tuple`` は例外です。 後者の2つには :py:mod:`typing` にエイリアスがあります： :py:class:`typing.Type` および :py:class:`typing.Tuple`。 ``InitVar`` は :py:mod:`dataclasses` からインポートする必要があります。 ``Callable`` は :py:mod:`typing` または :py:mod:`collections.abc` からインポートできます。 特殊形式はエイリアス化できます（例：``from typing import Literal as L``）、および修飾名で参照できます（例：``typing.Literal``）。 注釈や型式で許可されていない他の特殊形式もあります。 これには ``Generic``、``Protocol``、および ``TypedDict`` が含まれます。

* ``name`` として示される任意のリーフは、修飾名（つまり、``module '.' name`` または ``package '.' module '.' name``、任意のレベルのネストを持つ）でもかまいません。

* 括弧内のコメントは、文法で表現されていない追加の制限や構成の意味を簡単に説明しています。

.. _ `string-annotations`:

.. _`forward-references`:

文字列注釈
------------------------------------------------------------------------------------------

型ヒントが実行時に評価できない場合、その定義は後で解決される文字列リテラルとして表現できます。

これが一般的に発生する状況は、クラスが定義されているコンテナクラスの定義です。 たとえば、次のコード（単純な二分木の実装の開始）は機能しません::

  class Tree:
      def __init__(self, left: Tree, right: Tree):
          self.left = left
          self.right = right

これに対処するために、次のように書きます::

  class Tree:
      def __init__(self, left: 'Tree', right: 'Tree'):
          self.left = left
          self.right = right

文字列リテラルには有効な Python 式が含まれている必要があります（つまり、``compile(lit, '', 'eval')`` は有効なコードオブジェクトである必要があります）し、モジュールが完全にロードされた後にエラーなしで評価される必要があります。 それが評価されるローカルおよびグローバル名前空間は、同じ関数のデフォルト引数が評価される名前空間と同じである必要があります。

さらに、式は有効な型ヒントとして解析可能である必要があります。 つまり、:ref:`the expression grammar <expression-grammar>` のルールによって制約されます。

トリプルクォートが使用される場合、文字列は暗黙的に括弧で囲まれているかのように解析される必要があります。 これにより、文字列リテラル内で改行文字を使用できます::

    value: """
        int |
        str |
        list[Any]
    """

文字列リテラルを型ヒントの *一部* として使用することも許可されます。 たとえば::

    class Tree:
        ...
        def leaves(self) -> list['Tree']:
            ...

前方参照の一般的な使用例は、たとえば Django モデルがシグネチャで必要な場合です。 通常、各モデルは別々のファイルにあり、他のモデルを含む引数を持つメソッドがあります。 Python での循環インポートの方法のため、すべての必要なモデルを直接インポートすることはしばしば不可能です::

    # File models/a.py
    from models.b import B
    class A(Model):
        def foo(self, b: B): ...

    # File models/b.py
    from models.a import A
    class B(Model):
        def bar(self, a: A): ...

    # File main.py
    from models.a import A
    from models.b import B

main が最初にインポートされると仮定すると、これは models/a.py からインポートされる前に models/b.py の ``from models.a import A`` 行で ImportError が発生します。 解決策は、モジュールのみのインポートに切り替え、モデルをその _module_._class_ 名で参照することです::

    # File models/a.py
    from models import b
    class A(Model):
        def foo(self, b: 'b.B'): ...

    # File models/b.py
    from models import a
    class B(Model):
        def bar(self, a: 'a.A'): ...

    # File main.py
    from models.a import A
    from models.b import B

ジェネレータ関数とコルーチンの注釈
------------------------------------------------------------------------------------------

ジェネレータ関数の戻り値の型は、``typing.py`` モジュールによって提供されるジェネリック型 ``Generator[yield_type, send_type, return_type]`` によって注釈できます::

  def echo_round() -> Generator[int, float, str]:
      res = yield 0
      while res:
          res = yield round(res)
      return 'OK'

:pep:`492` で導入されたコルーチンは、通常の関数と同じ構文で注釈されます。 ただし、戻り値の型注釈はコルーチン型ではなく ``await`` 式の型に対応します::

  async def spam(ignored: int) -> str:
      return 'spam'

  async def foo() -> None:
      bar = await spam(42)  # type is str

ジェネリック ABC ``collections.abc.Coroutine`` を使用して、``send()`` および ``throw()`` メソッドをサポートする awaitable を指定できます。 型変数の分散と順序は ``Generator`` のものに対応し、``Coroutine[T_co, T_contra, V_co]`` です。 たとえば::

  from collections.abc import Coroutine
  c: Coroutine[list[str], str, int]
  ...
  x = c.send('hi')  # type is list[str]
  async def bar() -> None:
      x = await c  # type is int

ジェネリック ABC ``Awaitable``、``AsyncIterable``、および ``AsyncIterator`` は、より正確な型を指定できない状況で使用できます::

  def op() -> collections.abc.Awaitable[str]:
      if cond:
          return spam(42)
      else:
          return asyncio.Future(...)

.. _`annotating-methods`:

インスタンスメソッドとクラスメソッドの注釈
------------------------------------------------------------------------------------------

ほとんどの場合、クラスメソッドおよびインスタンスメソッドの最初の引数には注釈を付ける必要はなく、インスタンスメソッドの場合は含まれているクラスの型を持つと見なされ、クラスメソッドの場合は含まれているクラスオブジェクトに対応する型オブジェクト型を持つと見なされます。 さらに、インスタンスメソッドの最初の引数には型変数で注釈を付けることができます。 この場合、戻り値の型は同じ型変数を使用することができ、そのメソッドをジェネリック関数にします。 たとえば::

  T = TypeVar('T', bound='Copyable')
  class Copyable:
      def copy(self: T) -> T:
          # return a copy of self

  class C(Copyable): ...
  c = C()
  c2 = c.copy()  # type here should be C

同じことが、最初の引数の注釈に ``type[]`` を使用するクラスメソッドにも当てはまります::

  T = TypeVar('T', bound='C')
  class C:
      @classmethod
      def factory(cls: type[T]) -> T:
          # make a new instance of cls

  class D(C): ...
  d = D.factory()  # type here should be D

一部の型チェッカーは、この使用に制限を適用する場合があります。 たとえば、使用される型変数に適切な上限を要求するなどです（例を参照）。
