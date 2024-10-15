.. _`literal-types`:

リテラル
==========================================================================================

.. _`literal`:

``Literal``
------------------------------------------------------------------------------------------

（元々 :pep:`586` で指定されています。）

コアセマンティクス
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、リテラル型の基本的な動作を概説します。

コアの動作
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

リテラル型は、変数が特定の具体的な値を持つことを示します。 たとえば、変数 ``foo`` の型を ``Literal[3]`` と定義する場合、``foo`` は正確に ``3`` と等しくなければならず、他の値は許可されません。

型 ``T`` のメンバーである値 ``v`` が与えられた場合、型 ``Literal[v]`` は ``T`` のサブタイプです。 たとえば、``Literal[3]`` は ``int`` のサブタイプです。

親型のすべてのメソッドはリテラル型によって直接継承されます。 したがって、型 ``Literal[3]`` の変数 ``foo`` がある場合、``foo`` は ``int`` の ``__add__`` メソッドを継承するため、``foo + 5`` のような操作を行うことは安全です。 ``foo + 5`` の結果の型は ``int`` です。

この「継承」動作は、:ref:`NewTypes <newtype>` を処理する方法と同じです。

2 つのリテラルの同値性
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

2 つの型 ``Literal[v1]`` と ``Literal[v2]`` は、次の 2 つの条件が両方とも真である場合に同値です。

1. ``type(v1) == type(v2)``
2. ``v1 == v2``

たとえば、``Literal[20]`` と ``Literal[0x14]`` は同値です。 ただし、``Literal[0]`` と ``Literal[False]`` は同値ではありません。これは、ランタイムで ``0 == False`` が 'true' と評価されるにもかかわらず、``0`` は ``int`` 型であり、``False`` は ``bool`` 型であるためです。

リテラルの共用体の短縮
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

リテラルは 1 つ以上の値でパラメータ化されます。 リテラルが複数の値でパラメータ化される場合、それはこれらの型の共用体とまったく同等に扱われます。 つまり、``Literal[v1, v2, v3]`` は ``Literal[v1] | Literal[v2] | Literal[v3]`` と同等です。

このショートカットは、多くの異なるリテラルを受け入れる関数のシグネチャを記述する際に、より使いやすくするのに役立ちます。 たとえば、``open(...)`` のような関数::

   # 注: これは実際の型シグネチャの簡略化です。
   _PathType = str | bytes | int

   @overload
   def open(path: _PathType,
            mode: Literal["r", "w", "a", "x", "r+", "w+", "a+", "x+"],
            ) -> IO[str]: ...
   @overload
   def open(path: _PathType,
            mode: Literal["rb", "wb", "ab", "xb", "r+b", "w+b", "a+b", "x+b"],
            ) -> IO[bytes]: ...

   # ユーザーがリテラル型を使用していない場合のフォールバックオーバーロード
   @overload
   def open(path: _PathType, mode: str) -> IO[Any]: ...

提供される値はすべて同じ型のメンバーである必要はありません。 たとえば、``Literal[42, "foo", True]`` は合法な型です。

ただし、リテラルは少なくとも 1 つの型でパラメータ化する必要があります。 ``Literal[]`` や ``Literal`` のような型は違法です。

合法および違法なパラメータ化
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、合法な ``Literal[...]`` 型を正確に構成するもの、つまりどの値がパラメータとして使用できるか、使用できないかについて説明します。

簡単に言えば、``Literal[...]`` 型は 1 つ以上のリテラル式でパラメータ化され、それ以外は何もできません。

.. _literal-legal-parameters:

型チェック時の ``Literal`` の合法なパラメータ
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

``Literal`` はリテラルの整数、バイトおよびユニコード文字列、ブール値、列挙値、および ``None`` でパラメータ化できます。 したがって、次のすべてが合法です::

   Literal[26]
   Literal[0x1A]  # Literal[26] と完全に同等
   Literal[-4]
   Literal["hello world"]
   Literal[b"hello world"]
   Literal[u"hello world"]
   Literal[True]
   Literal[Color.RED]  # Color が何らかの列挙であると仮定
   Literal[None]

**注:** 型 ``None`` は単一の値によって占有されるため、型 ``None`` と ``Literal[None]`` は完全に同等です。 型チェッカーは ``Literal[None]`` を単に ``None`` に簡略化する場合があります。

``Literal`` は他のリテラル型や他のリテラル型への型エイリアスでもパラメータ化できます。 たとえば、次のようにすることができます::

    ReadOnlyMode         = Literal["r", "r+"]
    WriteAndTruncateMode = Literal["w", "w+", "wt", "w+t"]
    WriteNoTruncateMode  = Literal["r+", "r+t"]
    AppendMode           = Literal["a", "a+", "at", "a+t"]

    AllModes = Literal[ReadOnlyMode, WriteAndTruncateMode,
                       WriteNoTruncateMode, AppendMode]

この機能は、リテラル型の使用と再利用をより使いやすくするために設計されています。

**注:** 上記のルールの結果として、型チェッカーは次のような型もサポートすることが期待されます::

    Literal[Literal[Literal[1, 2, 3], "foo"], 5, None]

これは次の型と完全に同等である必要があります::

    Literal[1, 2, 3, "foo", 5, None]

...そして次の型とも同等です::

    Literal[1, 2, 3, "foo", 5] | None

**注:** ``Literal["foo"]`` のような文字列リテラル型は、通常の文字列リテラルがランタイムで行うのと同じ方法でバイトまたはユニコードをサブタイプ化する必要があります。

たとえば、Python 3 では、型 ``Literal["foo"]`` は ``Literal[u"foo"]`` と同等です。これは、Python 3 では ``"foo"`` が ``u"foo"`` と同等であるためです。

同様に、Python 2 では、型 ``Literal["foo"]`` は ``Literal[b"foo"]`` と同等です。 ただし、ファイルに ``from __future__ import unicode_literals`` インポートが含まれている場合は、``Literal[u"foo"]`` と同等です。

型チェック時の ``Literal`` の違法なパラメータ
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

次のパラメータは設計上意図的に禁止されています。

- ``Literal[3 + 4]`` や ``Literal["foo".replace("o", "b")]`` のような任意の式。

  - 理由: リテラル型は型付けエコシステムへの最小限の拡張を意図しており、型チェッカーが型内の潜在的な式を解釈することを要求することは複雑さを増しすぎます。

  - この結果として、``Literal[4 + 3j]`` や ``Literal[-4 + 2j]`` のような複素数も禁止されています。 一貫性のために、単一の複素数を含む ``Literal[4j]`` のようなリテラルも禁止されています。

  - このルールの唯一の例外は、整数に対する単項 ``-``（マイナス）および単項 ``+``（プラス）です。 ``Literal[-5]`` および ``Literal[+1]`` のような型は受け入れられます。

- 有効なリテラル型を含むタプル（例: ``Literal[(1, "foo", "bar")]``）。 ユーザーはこの型を ``tuple[Literal[1], Literal["foo"], Literal["bar"]]`` として表現することができます。 また、タプルは ``Literal[1, 2, 3]`` のショートカットと混同される可能性があります。

- ミュータブルなリテラルデータ構造（例: 辞書リテラル、リストリテラル、またはセットリテラル）。 リテラルは常に暗黙的に最終的であり、変更不可能です。 したがって、``Literal[{"a": "b", "c": "d"}]`` は違法です。

- その他の型（例: ``Literal[Path]`` や ``Literal[some_object_instance]``）は違法です。 これには型変数が含まれます。 ``T`` が型変数である場合、``Literal[T]`` は許可されません。 型変数は値ではなく型に対してのみ変化します。

次のものは簡単のために暫定的に禁止されています。 将来的に許可することを検討できます。

- 浮動小数点数（例: ``Literal[3.14]``）。 無限大や NaN のリテラルをクリーンに表現することは難しく、実際の API は浮動小数点数のパラメータに基づいて動作を変えることはほとんどありません。

- ``Any``（例: ``Literal[Any]``）。 ``Any`` は型であり、``Literal[...]`` は値のみを含むことを意図しています。 また、``Literal[Any]`` が実際に何を意味するのかも不明です。

ランタイムでのパラメータ
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型チェック時に ``Literal[...]`` が含むことができるパラメータのセットは非常に小さいですが、``typing.Literal`` の実際の実装はランタイムでチェックを行いません。 たとえば::

   def my_function(x: Literal[1 + 2]) -> int:
       return x * 3

   x: Literal = 3
   y: Literal[my_function] = my_function

型チェッカーはこのプログラムを拒否する必要があります。 3 つの ``Literal`` の使用はすべてこの仕様に従って無効です。 ただし、Python 自体はこのプログラムをエラーなしで実行する必要があります。

これは、将来的に ``Literal`` の使用範囲を拡大する場合の柔軟性を保持するための一部であり、部分的にはランタイムで違法なパラメータをすべて検出することが不可能であるためです。 たとえば、ランタイムで ``Literal[1 + 2]`` と ``Literal[3]`` を区別することは不可能です。

リテラル、列挙、および前方参照
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

1 つの潜在的な曖昧さは、リテラル文字列とリテラル列挙メンバーへの前方参照の間にあります。 たとえば、``Literal["Color.RED"]`` 型があるとします。 このリテラル型には文字列リテラルが含まれていますか、それとも ``Color.RED`` 列挙メンバーへの前方参照ですか？

このような場合、常にユーザーがリテラル文字列を構築しようとしたと仮定します。 ユーザーが前方参照を望む場合、リテラル型全体を文字列でラップする必要があります。 例: ``"Literal[Color.RED]"``。

型推論
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、リテラルと型推論に関するいくつかのルールと例を説明します。

後方互換性
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型チェッカーがリテラルのサポートを追加する場合、後方互換性を最大化する方法で行うことが重要です。 型チェッカーは、リテラルのサポートが追加された後も、以前に型チェックされていたコードが引き続き型チェックされることを最善の努力で保証する必要があります。

これは特に型推論を行う場合に重要です。 たとえば、``x = "blue"`` という文が与えられた場合、``x`` の推論された型は ``str`` ですか、それとも ``Literal["blue"]`` ですか？

1 つの単純な戦略は、常に式がリテラル型であると仮定することです。 したがって、上記の例では、``x`` は常に ``Literal["blue"]`` の型を持つことになります。 この単純な戦略はほぼ確実に破壊的すぎます。 以前は型チェックに合格していたプログラムが失敗する原因となります。 例::

    # 型チェッカーが 'var' の型を Literal[3] と推論し、
    # my_list の型を List[Literal[3]] と推論する場合...
    var = 3
    my_list = [var]

    # ...この呼び出しは型エラーになります。
    my_list.append(4)

この戦略が失敗するもう 1 つの例は、オブジェクトのフィールドを設定する場合です::

    class MyObject:
        def __init__(self) -> None:
            # 型チェッカーが MyObject.field の型を Literal[3] と推論する場合...
            self.field = 3

    m = MyObject()

    # ...この代入はもはや型チェックに合格しません
    m.field = 4

すべてのケースで互換性を維持する別の戦略は、明示的に注釈されない限り、式がリテラル型でないと常に仮定することです。 この戦略を使用する型チェッカーは、上記の最初の例では常に ``x`` が ``str`` 型であると推論します。

これは唯一の実行可能な戦略ではありません。 型チェッカーは、より洗練された推論技術を試すことができます。 特定の戦略は義務付けられていませんが、型チェッカーは後方互換性の重要性を念頭に置く必要があります。

リテラルコンテキストでの非リテラルの使用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

リテラル型は、追加の特別なケースなしで既存のサブタイピングルールに従います。 たとえば、次のようなプログラムは型安全です::

   def expects_str(x: str) -> None: ...
   var: Literal["foo"] = "foo"

   # 合法: Literal["foo"] は str のサブタイプです
   expects_str(var)

これにより、一般的に非リテラル型はリテラル型に :term:`assignable` ではありません。 たとえば::

   def expects_literal(x: Literal["foo"]) -> None: ...

   def runner(my_str: str) -> None:
       # 違法: str は Literal["foo"] に代入できません
       expects_literal(my_str)

**注:** ユーザーが API がリテラルと元の型の両方を受け入れることをサポートしたい場合（おそらくレガシーの目的で）、フォールバックオーバーロードを実装する必要があります。 :ref:`literalstring-overloads` を参照してください。

他の型および機能との相互作用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、リテラル型が他の既存の型とどのように相互作用するかについて説明します。

構造化データのインテリジェントなインデックス付け
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

リテラルは、タプル、NamedTuple、およびクラスのような構造化型に「インテリジェントにインデックス付け」するために使用できます。 （注: これは包括的なリストではありません。）

たとえば、型チェッカーは、整数キーを使用してタプルにインデックスを付けるときに正しい値の型を推論する必要があります。::

   a: Literal[0] = 0
   b: Literal[5] = 5

   some_tuple: tuple[int, str, List[bool]] = (3, "abc", [True, False])
   reveal_type(some_tuple[a])   # 明らかにされた型は 'int' です
   some_tuple[b]                # エラー: 5 はタプルの有効なインデックスではありません

getattr のような関数を使用する場合も同様の動作が期待されます。::

   class Test:
       def __init__(self, param: int) -> None:
           self.myfield = param

       def mymethod(self, val: int) -> str: ...

   a: Literal["myfield"]  = "myfield"
   b: Literal["mymethod"] = "mymethod"
   c: Literal["blah"]     = "blah"

   t = Test()
   reveal_type(getattr(t, a))  # 明らかにされた型は 'int' です
   reveal_type(getattr(t, b))  # 明らかにされた型は 'Callable[[int], str]' です
   getattr(t, c)               # エラー: Test に 'blah' という名前の属性はありません

**注:** 上記の変数宣言をよりコンパクトに表現する方法については、 :ref:`literal-final-interactions` を参照してください。

オーバーロードとの相互作用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

リテラル型とオーバーロードは特別な方法で相互作用する必要はありません。 既存のルールで問題ありません。

ただし、型チェッカーがサポートする必要がある重要なユースケースの 1 つは、ユーザーがリテラル型を使用していない場合にフォールバックを使用する機能です。 たとえば、``open`` を考えてみましょう。::

   _PathType = str | bytes | int

   @overload
   def open(path: _PathType,
            mode: Literal["r", "w", "a", "x", "r+", "w+", "a+", "x+"],
            ) -> IO[str]: ...
   @overload
   def open(path: _PathType,
            mode: Literal["rb", "wb", "ab", "xb", "r+b", "w+b", "a+b", "x+b"],
            ) -> IO[bytes]: ...

   # ユーザーがリテラル型を使用していない場合のフォールバックオーバーロード
   @overload
   def open(path: _PathType, mode: str) -> IO[Any]: ...

``open`` のシグネチャを最初の 2 つのオーバーロードだけを使用するように変更すると、リテラル文字列式を渡さないコードが壊れます。 たとえば、次のようなコードが壊れます。::

   mode: str = pick_file_mode(...)
   with open(path, mode) as f:
       # f はここで IO[Any] 型である必要があります

もう少し広く言えば、typeshed の既存の API にリテラル型を追加するたびに、後方互換性を維持するために常にフォールバックオーバーロードを含めることを義務付けます。

ジェネリックとの相互作用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

リテラル型は型であり、型が期待される場所ならどこでも使用できます。

たとえば、ジェネリック関数やクラスをリテラル型でパラメータ化することは合法です。::

   A = TypeVar('A', bound=int)
   B = TypeVar('B', bound=int)
   C = TypeVar('C', bound=int)

   # Matrix[row, column] の簡略化された定義
   class Matrix(Generic[A, B]):
       def __add__(self, other: Matrix[A, B]) -> Matrix[A, B]: ...
       def __matmul__(self, other: Matrix[B, C]) -> Matrix[A, C]: ...
       def transpose(self) -> Matrix[B, A]: ...

   foo: Matrix[Literal[2], Literal[3]] = Matrix(...)
   bar: Matrix[Literal[3], Literal[7]] = Matrix(...)

   baz = foo @ bar
   reveal_type(baz)  # 明らかにされた型は 'Matrix[Literal[2], Literal[7]]' です

同様に、リテラル型を含む制限や境界を持つ型変数を構築することも合法です。::

   T = TypeVar('T', Literal["a"], Literal["b"], Literal["c"])
   S = TypeVar('S', bound=Literal["foo"])

...ただし、リテラルの上限を持つ型変数を構築することがいつ役立つかは不明です。 たとえば、上記の例の ``S`` 型変数は本質的に無意味です。 ``S = Literal["foo"]`` を使用することで同等の動作を得ることができます。

**注:** リテラル型とジェネリックは意図的に非常に基本的で限定的な方法でのみ相互作用します。 特に、数値操作や numpy スタイルの操作を大量に含むコードを型チェックしたいライブラリは、ここで説明するリテラル型がニーズに対して不十分であるとほぼ確実に感じるでしょう。

列挙と網羅性チェックとの相互作用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型チェッカーは、列挙のような限られた数のバリアントを持つリテラル型を扱う場合に網羅性チェックを実行できる必要があります。 たとえば、型チェッカーは、``Status`` 列挙の 3 つの値がすべて使い果たされたため、最後の ``else`` 文が ``str`` 型である必要があることを推論できる必要があります。::

    class Status(Enum):
        SUCCESS = 0
        INVALID_DATA = 1
        FATAL_ERROR = 2

    def parse_status(s: str | Status) -> None:
        if s is Status.SUCCESS:
            print("Success!")
        elif s is Status.INVALID_DATA:
            print("The given data is invalid because...")
        elif s is Status.FATAL_ERROR:
            print("Unexpected fatal error...")
        else:
            # 's' はすべての他のオプションが使い果たされたため 'str' 型である必要があります
            print("Got custom status: " + s)

ここで、``Status`` 列挙は ``Literal[Status.SUCCESS, Status.INVALID_DATA, Status.FATAL_ERROR]`` とほぼ同等と見なされ、``s`` の型がそれに応じて絞り込まれます。

絞り込みとの相互作用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型チェッカーは、上記のセクションで説明されているものを超えて、列挙および非列挙リテラル型に対して追加の分析をオプションで実行する場合があります。

たとえば、包含や等価性チェックに基づいて絞り込みを行うことが有用な場合があります。::

   def parse_status(status: str) -> None:
       if status in ("MALFORMED", "ABORTED"):
           # 型チェッカーは 'status' を Literal["MALFORMED", "ABORTED"] 型に絞り込むことができます。
           return expects_bad_status(status)

       # 同様に、型チェッカーは 'status' を Literal["PENDING"] 型に絞り込むことができます。
       if status == "PENDING":
           expects_pending_status(status)

リテラルブールを含む式を考慮して絞り込みを行うことも有用です。 たとえば、``Literal[True]``、``Literal[False]``、およびオーバーロードを組み合わせて「カスタム型ガード」を構築できます。::

   @overload
   def is_int_like(x: int | list[int]) -> Literal[True]: ...
   @overload
   def is_int_like(x: object) -> bool: ...
   def is_int_like(x): ...

   vector: list[int] = [1, 2, 3]
   if is_int_like(vector):
       vector.append(3)
   else:
       vector.append("bad")   # このブランチは到達不可能であると推論されます

   scalar: int | str
   if is_int_like(scalar):
       scalar += 3      # 型チェック: 'scalar' の型は 'int' に絞り込まれます
   else:
       scalar += "foo"  # 型チェック: 'scalar' の型は 'str' に絞り込まれます

.. _literal-final-interactions:

Final との相互作用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

``Final`` 修飾子を使用して、変数や属性が再代入できないことを宣言できます。::

    foo: Final = 3
    foo = 4           # エラー: 'foo' は Final と宣言されています

上記の例では、``foo`` が常に正確に ``3`` と等しいことがわかります。 型チェッカーはこの情報を使用して、``foo`` が ``Literal[3]`` を期待するコンテキストで使用することが有効であると推論できます。::

    def expects_three(x: Literal[3]) -> None: ...

    expects_three(foo)  # 型チェックに合格します。'foo' は Final であり、3 と等しいためです。

``Final`` 修飾子は、変数が*実質的にリテラル*であることを宣言するための省略形として機能します。

型チェッカーはこのショートカットをサポートすることが期待されます。 具体的には、``var: Final = value`` の形式の変数または属性の代入があり、``value`` が ``Literal[...]`` の有効なパラメータである場合、型チェッカーは ``var`` が ``Literal[value]`` を期待するコンテキストで使用できることを理解する必要があります。

型チェッカーは、Final の他の使用法を理解する義務はありません。 たとえば、次のプログラムが型チェックに合格するかどうかは指定されていません。::

    # 注: 代入は正確に 'var: Final = value' の形式と一致しません。
    bar1: Final[int] = 3
    expects_three(bar1)  # 型チェッカーによって受け入れられる場合と受け入れられない場合があります。

    # 注: "Literal[1 + 2]" は合法な型ではありません。
    bar2: Final = 1 + 2
    expects_three(bar2)  # 型チェッカーによって受け入れられる場合と受け入れられない場合があります。

.. _`literalstring`:

``LiteralString``
------------------------------------------------------------------------------------------

（元々 :pep:`675` で指定されています。）

``LiteralString`` の有効な場所
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``LiteralString`` は他の型が使用できる場所で使用できます。::

    variable_annotation: LiteralString

    def my_function(literal_string: LiteralString) -> LiteralString: ...

    class Foo:
        my_attribute: LiteralString

    type_argument: List[LiteralString]

    T = TypeVar("T", bound=LiteralString)

共用体の ``Literal`` 型内にネストすることはできません。::

    bad_union: Literal["hello", LiteralString]  # 許可されません
    bad_nesting: Literal[LiteralString]  # 許可されません

型推論
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``LiteralString`` の推論
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

任意のリテラル文字列型は ``LiteralString`` に代入できます。 たとえば、``x: LiteralString = "foo"`` は有効です。なぜなら、``"foo"`` は ``Literal["foo"]`` 型と推論されるためです。

次の場合にも ``LiteralString`` を推論します。

+ 加算: ``x`` と ``y`` の両方の型が ``LiteralString`` に代入できる場合、``x + y`` の型は ``LiteralString`` です。

+ 結合: ``sep`` の型が ``LiteralString`` に代入でき、``xs`` の型が ``Iterable[LiteralString]`` に代入できる場合、``sep.join(xs)`` の型は ``LiteralString`` です。

+ インプレース加算: ``s`` の型が ``LiteralString`` であり、``x`` の型が ``LiteralString`` に代入できる場合、``s += x`` は ``s`` の型を ``LiteralString`` として保持します。

+ 文字列フォーマット: f 文字列の型は、その構成要素の式がリテラル文字列である場合にのみ ``LiteralString`` です。 ``s.format(...)`` は、``s`` と引数の型が ``LiteralString`` に代入できる場合にのみ ``LiteralString`` に代入できます。

他のすべての場合、構成される値の 1 つ以上が非リテラル型 ``str`` を持つ場合、型の構成は ``str`` 型を持ちます。 たとえば、``s`` の型が ``str`` である場合、``"hello" + s`` の型は ``str`` です。 これは型チェッカーの既存の動作と一致します。

``LiteralString`` は ``str`` 型に代入できます。 それは ``str`` からすべてのメソッドを継承します。 したがって、``LiteralString`` 型の変数 ``s`` がある場合、``s.startswith("hello")`` と書くことは安全です。

一部の型チェッカーは、等価性チェックを行うときに文字列の型を絞り込みます。::

    def foo(s: str) -> None:
        if s == "bar":
            reveal_type(s)  # => Literal["bar"]

if ブロック内でのこのような絞り込まれた型も ``LiteralString`` に代入できます。なぜなら、その型は ``Literal["bar"]`` だからです。

例
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

以下の例を参照して、上記のルールを明確にしてください。::

    literal_string: LiteralString
    s: str = literal_string  # OK

    literal_string: LiteralString = s  # エラー: LiteralString が期待されましたが、str が得られました。
    literal_string: LiteralString = "hello"  # OK

リテラル文字列の加算::

    def expect_literal_string(s: LiteralString) -> None: ...

    expect_literal_string("foo" + "bar")  # OK
    expect_literal_string(literal_string + "bar")  # OK

    literal_string2: LiteralString
    expect_literal_string(literal_string + literal_string2)  # OK

    plain_string: str
    expect_literal_string(literal_string + plain_string)  # 許可されません。

リテラル文字列を使用した結合::

    expect_literal_string(",".join(["foo", "bar"]))  # OK
    expect_literal_string(literal_string.join(["foo", "bar"]))  # OK
    expect_literal_string(literal_string.join([literal_string, literal_string2]))  # OK

    xs: List[LiteralString]
    expect_literal_string(literal_string.join(xs)) # OK
    expect_literal_string(plain_string.join([literal_string, literal_string2]))
    # 区切り文字の型が 'str' であるため許可されません。

リテラル文字列を使用したインプレース加算::

    literal_string += "foo"  # OK
    literal_string += literal_string2  # OK
    literal_string += plain_string # 許可されません

リテラル文字列を使用したフォーマット文字列::

    literal_name: LiteralString
    expect_literal_string(f"hello {literal_name}")
    # それがリテラル文字列から構成されているため OK です。

    expect_literal_string("hello {}".format(literal_name))  # OK

    expect_literal_string(f"hello")  # OK

    username: str
    expect_literal_string(f"hello {username}")
    # 許可されません。フォーマット文字列は 'username' から構成されており、
    # その型は 'str' です。

    expect_literal_string("hello {}".format(username))  # 許可されません

リテラル整数などの他のリテラル型は ``LiteralString`` に代入できません。::

    some_int: int
    expect_literal_string(some_int)  # エラー: LiteralString が期待されましたが、int が得られました。

    literal_one: Literal[1] = 1
    expect_literal_string(literal_one)  # エラー: LiteralString が期待されましたが、Literal[1] が得られました。

リテラル文字列に関数を呼び出すことができます。::

    def add_limit(query: LiteralString) -> LiteralString:
        return query + " LIMIT = 1"

    def my_query(query: LiteralString, user_id: str) -> None:
        sql_connection().execute(add_limit(query), (user_id,))  # OK

条件文と式は期待通りに動作します。::

    def return_literal_string() -> LiteralString:
        return "foo" if condition1() else "bar"  # OK

    def return_literal_str2(literal_string: LiteralString) -> LiteralString:
        return "foo" if condition1() else literal_string  # OK

    def return_literal_str3() -> LiteralString:
        if condition1():
            result: Literal["foo"] = "foo"
        else:
            result: LiteralString = "bar"

        return result  # OK

型変数とジェネリックとの相互作用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型変数は ``LiteralString`` にバインドできます。::

    from typing import Literal, LiteralString, TypeVar

    TLiteral = TypeVar("TLiteral", bound=LiteralString)

    def literal_identity(s: TLiteral) -> TLiteral:
        return s

    hello: Literal["hello"] = "hello"
    y = literal_identity(hello)
    reveal_type(y)  # => Literal["hello"]

    s: LiteralString
    y2 = literal_identity(s)
    reveal_type(y2)  # => LiteralString

    s_error: str
    literal_identity(s_error)
    # エラー: LiteralString にバインドされた TLiteral が期待されましたが、str が得られました。

``LiteralString`` はジェネリッククラスの型引数として使用できます。::

    class Container(Generic[T]):
        def __init__(self, value: T) -> None:
            self.value = value

    literal_string: LiteralString = "hello"
    x: Container[LiteralString] = Container(literal_string)  # OK

    s: str
    x_error: Container[LiteralString] = Container(s)  # 許可されません

``List`` のような標準コンテナは期待通りに動作します。::

    xs: List[LiteralString] = ["foo", "bar", "baz"]

.. _literalstring-overloads:

オーバーロードとの相互作用
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

リテラル文字列とオーバーロードは特別な方法で相互作用する必要はありません。 既存のルールで問題ありません。 ``LiteralString`` は、特定の ``Literal["foo"]`` 型が一致しない場合のフォールバックオーバーロードとして使用できます。::

    @overload
    def foo(x: Literal["foo"]) -> int: ...
    @overload
    def foo(x: LiteralString) -> bool: ...
    @overload
    def foo(x: str) -> str: ...

    x1: int = foo("foo")  # 最初のオーバーロード。
    x2: bool = foo("bar")  # 2 番目のオーバーロード。
    s: str
    x3: str = foo(s)  # 3 番目のオーバーロード。
