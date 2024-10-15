.. _`typed-dictionaries`:

型付き辞書
==========================================================================================

.. _`typeddict`:

TypedDict
------------------------------------------------------------------------------------------

(元々は :pep:`589` で指定されていました。)

TypedDict 型は特定のセットの文字列キーを持ち、各有効なキーに対して特定の値の型を持つ辞書オブジェクトを表します。 各文字列キーは必須 (存在しなければならない) または非必須 (存在する必要はない) のいずれかです。

TypedDict 型を定義する方法は 2 つあります。 1 つ目はクラスベースの構文を使用します。 2 つ目は代替の代入ベースの構文で、後方互換性のために提供されています。 この機能を古い Python バージョンにバックポートできるようにするためです。 理由は :pep:`484` が Python 2.7 のためにコメントベースの注釈構文をサポートする理由と似ています: 型ヒントは特に大規模な既存のコードベースに役立ち、これらはしばしば古い Python バージョンで実行する必要があります。 2 つの構文オプションは ``typing.NamedTuple`` がサポートする構文のバリエーションと並行しています。 その他の機能には TypedDict の継承と全体性 (キーが必須かどうかを指定する) が含まれます。

このセクションでは、TypedDict オブジェクトを含む型チェック操作を型チェッカーがサポートする方法の概要も提供します。 :pep:`484` と同様に、この議論は意図的に曖昧にされています。 これは、さまざまな型チェックアプローチの実験を許可するためです。 特に、:term:`assignability <assignable>` は :term:`structural` である必要があります。 より具体的な TypedDict 型は、これらの間に継承関係がなくても、より一般的な TypedDict 型に割り当て可能である必要があります。


クラスベースの構文
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

TypedDict 型は、唯一の基底クラスとして ``typing.TypedDict`` を使用してクラス定義構文を使用して定義できます::

    from typing import TypedDict

    class Movie(TypedDict):
        name: str
        year: int

``Movie`` は 2 つの項目を持つ TypedDict 型です: ``'name'`` (型 ``str``) と ``'year'`` (型 ``int``)。

型チェッカーは、クラスベースの TypedDict 定義の本体が次のルールに従っていることを検証する必要があります:

* クラス本体には、オプションでドキュメント文字列が前に付いた ``key: value_type`` の形式の項目定義のみが含まれている必要があります。 項目定義の構文は属性注釈と同じですが、初期化子はなく、キー名は属性名ではなくキーの文字列値を指します。

* 一貫性のために、クラスベースの ``NamedTuple`` 構文と同様に、クラスベースの構文では型コメントを使用できません。 代わりに、`代替構文`_ は後方互換性のための代替の代入ベースの構文を提供します。

* 文字列リテラルの前方参照は値の型で有効です。

* メソッドは許可されていません。 これは、TypedDict オブジェクトのランタイム型は常に ``dict`` であり、``dict`` のサブクラスにはならないためです。

* メタクラスの指定は許可されていません。

* TypedDict は、基底クラスに ``Generic[T]`` を追加することでジェネリックにすることができます (または、Python 3.12 以降では、ジェネリッククラスの新しい構文を使用します)。

本体に ``pass`` のみを含めることで空の TypedDict を作成できます (ドキュメント文字列がある場合は ``pass`` を省略できます)::

    class EmptyDict(TypedDict):
        pass


TypedDict 型の使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

次に、型 ``Movie`` を使用する方法の例を示します::

    movie: Movie = {'name': 'Blade Runner',
                    'year': 1982}

明示的な ``Movie`` 型の注釈が一般的に必要です。 これは、後方互換性のために、型チェッカーによって通常の辞書型が仮定される可能性があるためです。 型チェッカーが構築された辞書オブジェクトが TypedDict であると推論できる場合、明示的な注釈を省略できます。 典型的な例は、関数引数としての辞書オブジェクトです。 この例では、型チェッカーは辞書引数が TypedDict として理解されるべきであると推論することが期待されます::

    def record_movie(movie: Movie) -> None: ...

    record_movie({'name': 'Blade Runner', 'year': 1982})

型チェッカーが辞書表示を TypedDict として扱うべき別の例は、以前に宣言された TypedDict 型の変数への代入です::

    movie: Movie
    ...
    movie = {'name': 'Blade Runner', 'year': 1982}

``movie`` に対する操作は静的型チェッカーによってチェックできます::

    movie['director'] = 'Ridley Scott'  # エラー: 無効なキー 'director'
    movie['year'] = '1982'  # エラー: 無効な値の型 ("int" が期待される)

次のコードは拒否されるべきです。 ``'title'`` は有効なキーではなく、``'name'`` キーが欠落しているためです::

    movie2: Movie = {'title': 'Blade Runner',
                     'year': 1982}

作成された TypedDict 型オブジェクトは実際のクラスオブジェクトではありません。 型チェッカーが許可することが期待される型の唯一の使用法は次のとおりです:

* 型注釈や型エイリアス、キャストのターゲット型など、任意の型ヒントが有効なコンテキストで使用できます。

* TypedDict 項目に対応するキーワード引数を持つ呼び出し可能オブジェクトとして使用できます。 非キーワード引数は許可されていません。 例::

      m = Movie(name='Blade Runner', year=1982)

  呼び出されたとき、TypedDict 型オブジェクトはランタイムで通常の辞書オブジェクトを返します::

      print(type(m))  # <class 'dict'>

* 基底クラスとして使用できますが、派生 TypedDict を定義する場合のみです。 これは以下で詳しく説明します。

特に、TypedDict 型オブジェクトは ``isinstance()`` テスト (例: ``isinstance(d, Movie)``) で使用できません。 その理由は、辞書項目値の型をチェックする既存のサポートがないためです。 ``isinstance()`` は多くの型で機能しません。 これには、``list[str]`` のような一般的な型も含まれます。 これは次のような場合に必要です::

    class Strings(TypedDict):
        items: list[str]

    print(isinstance({'items': [1]}, Strings))    # False であるべき
    print(isinstance({'items': ['x']}, Strings))  # True であるべき

上記の使用例はサポートされていません。 これは、``isinstance()`` が ``list[str]`` に対してサポートされていないことと一致しています。


継承
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

TypedDict 型は、クラスベースの構文を使用して 1 つ以上の TypedDict 型から継承することができます。 この場合、``TypedDict`` 基底クラスは含めないでください。 例::

    class BookBasedMovie(Movie):
        based_on: str

これで ``BookBasedMovie`` には ``name``、``year``、および ``based_on`` のキーがあります。 これは次の定義と同等です。 TypedDict 型は :term:`structural` :term:`assignability <assignable>` を使用するためです::

    class BookBasedMovie(TypedDict):
        name: str
        year: int
        based_on: str

次に、複数の継承の例を示します::

    class X(TypedDict):
        x: int

    class Y(TypedDict):
        y: str

    class XYZ(X, Y):
        z: bool

TypedDict ``XYZ`` には 3 つの項目があります: ``x`` (型 ``int``)、``y`` (型 ``str``)、および ``z`` (型 ``bool``)。

TypedDict は、TypedDict 型と ``Generic`` 以外の非 TypedDict 基底クラスの両方から継承することはできません。

TypedDict クラスの継承に関する追加の注意事項:

* サブクラスで親 TypedDict クラスのフィールド型を変更することは許可されていません。 例::

   class X(TypedDict):
      x: str

   class Y(X):
      x: int  # 型チェックエラー: TypedDict フィールド "x" を上書きできません

  上記の例では、TypedDict クラスの注釈はキー ``x`` に対して型 ``str`` を返します::

   print(Y.__annotations__)  # {'x': <class 'str'>}


* 同じ名前のフィールドに対して競合する型を持つことは許可されていません::

   class X(TypedDict):
      x: int

   class Y(TypedDict):
      x: str

   class XYZ(X, Y):  # 型チェックエラー: マージ中に TypedDict フィールド "x" を上書きできません
      xyz: bool


全体性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

デフォルトでは、TypedDict のすべてのキーが存在する必要があります。 *全体性* を指定することでこれを上書きすることができます。 クラスベースの構文を使用してこれを行う方法は次のとおりです::

    class Movie(TypedDict, total=False):
        name: str
        year: int

これは、``Movie`` TypedDict が任意のキーを省略できることを意味します。 したがって、次のように有効です::

    m: Movie = {}
    m2: Movie = {'year': 2015}

型チェッカーは、``total`` 引数の値としてリテラル ``False`` または ``True`` のみをサポートすることが期待されます。 ``True`` はデフォルトであり、クラス本体で定義されたすべての項目を必須にします。

全体性フラグは、TypedDict 定義の本体で定義された項目にのみ適用されます。 継承された項目は影響を受けず、それらが定義された TypedDict 型の全体性を使用します。 これにより、単一の TypedDict 型で必須キーと非必須キーの組み合わせを持つことができます。 代わりに、個々の項目を必須または非必須としてマークするために ``Required`` および ``NotRequired`` (以下を参照) を使用できます。

.. _typeddict-functional-syntax:

代替構文
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、:pep:`526` で導入された変数定義構文をサポートしない 3.5 や 2.7 などの古い Python バージョンにバックポートできる代替構文を提供します。 これは、名前付きタプルを定義するための従来の構文に似ています::

    Movie = TypedDict('Movie', {'name': str, 'year': int})

代替構文を使用して全体性を指定することもできます::

    Movie = TypedDict('Movie',
                      {'name': str, 'year': int},
                      total=False)

意味論はクラスベースの構文と同等です。 ただし、この構文は継承をサポートしていません。 これの動機は、後方互換性のある構文をできるだけシンプルに保ちながら、最も一般的な使用例をカバーすることです。

型チェッカーは、``TypedDict`` の 2 番目の引数として辞書表示式のみを受け入れることが期待されます。 特に、辞書オブジェクトを参照する変数は、実装を簡素化するためにサポートする必要はありません。


割り当て可能性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

まず、任意の TypedDict 型は ``Mapping[str, object]`` に :term:`assignable` です。

次に、TypedDict 型 ``B`` は、次の 2 つの条件が満たされている場合に限り、TypedDict ``A`` に :term:`assignable` です:

* ``A`` の各キーに対して、``B`` は対応するキーを持ち、``B`` の対応する値の型は ``A`` の値の型と :term:`consistent` です。

* ``B`` の各必須キーに対して、対応するキーは ``A`` で必須です。 ``B`` の各非必須キーに対して、対応するキーは ``A`` で非必須です。

議論:

* 値の型は不変に動作します。 これは、TypedDict オブジェクトが変更可能であるためです。 これは、``List`` や ``Dict`` などの変更可能なコンテナ型と同様です。 これが関連する例::

      class A(TypedDict):
          x: int | None

      class B(TypedDict):
          x: int

      def f(a: A) -> None:
          a['x'] = None

      b: B = {'x': 0}
      f(b)  # 型チェックエラー: 'B' は 'A' に割り当て可能ではありません
      b['x'] + 1  # ランタイムエラー: None + 1

* 必須キーを持つ TypedDict 型は、同じキーが非必須キーである TypedDict 型に :term:`assignable` ではありません。 これは、後者がキーを削除できるためです。 これが関連する例::

      class A(TypedDict, total=False):
          x: int

      class B(TypedDict):
          x: int

      def f(a: A) -> None:
          del a['x']

      b: B = {'x': 0}
      f(b)  # 型チェックエラー: 'B' は 'A' に割り当て可能ではありません
      b['x'] + 1  # ランタイム KeyError: 'x'

* キー ``'x'`` を持たない TypedDict 型 ``A`` は、非必須キー ``'x'`` を持つ TypedDict 型に :term:`assignable` ではありません。 これは、ランタイムでキー ``'x'`` が存在し、:term:`inconsistent <consistent>` 型を持つ可能性があるためです (これは :term:`structural` assignability によって ``A`` を通じて表示されない場合があります)。 例::

      class A(TypedDict, total=False):
          x: int
          y: int

      class B(TypedDict, total=False):
          x: int

      class C(TypedDict, total=False):
          x: int
          y: str

       def f(a: A) -> None:
           a['y'] = 1

       def g(b: B) -> None:
           f(b)  # 型チェックエラー: 'B' は 'A' に割り当て可能ではありません

       c: C = {'x': 0, 'y': 'foo'}
       g(c)
       c['y'] + 'bar'  # ランタイムエラー: int + str

* TypedDict は、任意の ``Dict[...]`` 型に :term:`assignable` ではありません。 これは、辞書型が破壊的な操作を許可するためです。 これには ``clear()`` も含まれます。 また、任意のキーを設定することも許可されており、これにより型の安全性が損なわれる可能性があります。 例::

      class A(TypedDict):
          x: int

      class B(A):
          y: str

      def f(d: Dict[str, int]) -> None:
          d['y'] = 0

      def g(a: A) -> None:
          f(a)  # 型チェックエラー: 'A' は Dict[str, int] に割り当て可能ではありません

      b: B = {'x': 0, 'y': 'foo'}
      g(b)
      b['y'] + 'bar'  # ランタイムエラー: int + str

* すべての ``int`` 値を持つ TypedDict は、``Mapping[str, int]`` に :term:`assignable` ではありません。 これは、:term:`structural` assignability によって型を通じて表示されない追加の非 ``int`` 値が存在する可能性があるためです。 これらは、たとえば ``Mapping`` の ``values()`` および ``items()`` メソッドを使用してアクセスできます。 例::

      class A(TypedDict):
          x: int

      class B(TypedDict):
          x: int
          y: str

      def sum_values(m: Mapping[str, int]) -> int:
          n = 0
          for v in m.values():
              n += v  # ランタイムエラー
          return n

      def f(a: A) -> None:
          sum_values(a)  # エラー: 'A' は Mapping[str, int] に割り当て可能ではありません

      b: B = {'x': 0, 'y': 'foo'}
      f(b)


サポートされている操作とサポートされていない操作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

型チェッカーは、TypedDict オブジェクトに対するほとんどの ``dict`` 操作の制限された形式をサポートする必要があります。 指針となる原則は、ランタイムの型安全性を侵害する可能性がある操作を型チェッカーが拒否するべきであるということです。 ここでは、回避するための最も重要な型安全性の違反のいくつかを示します:

1. 必須キーが欠落している。

2. 値が無効な型を持っている。

3. TypedDict 型に定義されていないキーが追加される。

キーがリテラルでない場合は一般的に拒否されるべきです。 これは、型チェック中にその値が不明であり、上記の違反のいくつかを引き起こす可能性があるためです。 (`Final 値とリテラル型の使用`_ は、これを最終名とリテラル型をカバーするように一般化します。)

キーが存在することが知られていない場合の使用は、ランタイム型エラーを生成しない場合でもエラーとして報告されるべきです。 これらはしばしば間違いであり、:term:`structural` :term:`assignability <assignable>` が特定の項目の型を隠す場合、無効な型の値を挿入する可能性があります。 たとえば、``d['x'] = 1`` は、``'x'`` が ``d`` の有効なキーでない場合、型チェックエラーを生成するべきです (これは TypedDict 型であると仮定されます)。

TypedDict オブジェクトの構築に含まれる余分なキーもキャッチされるべきです。 この例では、``director`` キーは ``Movie`` に定義されておらず、型チェッカーからエラーが生成されることが期待されます::

    m: Movie = dict(
        name='Alien',
        year=1979,
        director='Ridley Scott')  # エラー: 予期しないキー 'director'

型チェッカーは、次の操作を TypedDict オブジェクトに対して安全でないとして拒否するべきです。 これらは通常の辞書に対しては有効です:

* 任意の ``str`` キー (文字列リテラルや既知の文字列値を持つ他の式ではなく) を使用した操作は一般的に拒否されるべきです。 これは、項目の設定などの破壊的な操作と、サブスクリプション式などの読み取り専用操作の両方を含みます。 上記のルールの例外として、``d.get(e)`` および ``e in d`` は、任意の ``str`` 型の式 ``e`` に対して TypedDict オブジェクトに対して許可されるべきです。 動機は、これらが安全であり、TypedDict オブジェクトを調査するのに役立つ可能性があるためです。 ``d.get(e)`` の静的型は、文字列値が静的に決定できない場合、``object`` であるべきです。

* ``clear()`` は、必須キーを削除する可能性があるため安全ではありません。 これには、:term:`structural` :term:`assignability <assignable>` によって直接表示されないキーも含まれます。 ``popitem()`` も同様に安全ではありません。 すべての既知のキーが必須でない場合 (``total=False``) でも同様です。

* ``del obj['key']`` は、``'key'`` が非必須キーでない限り拒否されるべきです。

型チェッカーは、キー ``'x'`` が必須でない場合でも、``d['x']`` を使用して項目を読み取ることを許可する場合があります。 代わりに、``d.get('x')`` または明示的な ``'x' in d`` チェックを要求する代わりにです。 動機は、キーの存在を追跡することは完全に一般的に実装するのが難しいことであり、これを許可しないと既存のコードに多くの変更が必要になる可能性があるためです。

正確な型チェックルールは各型チェッカーが決定するものです。 一部のケースでは、潜在的に安全でない操作が受け入れられる場合があります。 これは、代替が慣用的なコードに対して誤検知エラーを生成する場合です。


Final 値とリテラル型の使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

型チェッカーは、TypedDict オブジェクトに対する操作で文字列値を持つ :ref:`final names <uppercase-final>` を文字列リテラルの代わりに使用することを許可するべきです。 たとえば、これは有効です::

   YEAR: Final = 'year'

   m: Movie = {'name': 'Alien', 'year': 1979}
   years_since_epoch = m[YEAR] - 1970

同様に、適切な :ref:`literal type <literal>` を持つ式をリテラル値の代わりに使用できます::

   def get_value(movie: Movie,
                 key: Literal['year', 'name']) -> int | str:
       return movie[key]

型チェッカーは、TypedDict 型定義でキーを指定するために実際の文字列リテラルのみをサポートすることが期待されます。 また、TypedDict 定義で全体性を指定するためにブールリテラルのみを使用できます。 動機は、型宣言を自己完結型にし、型チェッカーの実装を簡素化することです。


後方互換性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

後方互換性を維持するために、型チェッカーは、プログラマーがこれを望んでいることが十分に明確でない限り、TypedDict 型を推論しないべきです。 確信が持てない場合は、通常の辞書型が推論されるべきです。 そうしないと、TypedDict サポートが型チェッカーに追加されると、型チェックエラーなしで型チェックされる既存のコードがエラーを生成し始める可能性があります。 これは、TypedDict 型が辞書型よりも制限が厳しいためです。 特に、辞書型のサブタイプではありません。

.. _`required-notrequired`:

``Required`` および ``NotRequired``
------------------------------------------------------------------------------------------

(元々は :pep:`655` で指定されていました。)

.. _`required`:

``typing.Required`` :term:`type qualifier` は、TypedDict 定義で宣言された変数が必須キーであることを示すために使用されます::

   class Movie(TypedDict, total=False):
       title: Required[str]
       year: int

.. _`notrequired`:

さらに、``typing.NotRequired`` :term:`type qualifier` は、TypedDict 定義で宣言された変数が存在する可能性のあるキーであることを示すために使用されます::

   class Movie(TypedDict):  # 暗黙的に total=True
       title: str
       year: NotRequired[int]

``Required[]`` または ``NotRequired[]`` を TypedDict の項目以外の場所で使用することはエラーです。 型チェッカーはこの制限を強制する必要があります。

必要に応じて、冗長であっても ``Required[]`` および ``NotRequired[]`` を使用することが有効です::

   class Movie(TypedDict):
       title: Required[str]  # 冗長
       year: NotRequired[int]

同時に ``Required[]`` と ``NotRequired[]`` を使用することはエラーです::

   class Movie(TypedDict):
       title: str
       year: NotRequired[Required[int]]  # エラー

型チェッカーはこの制限を強制する必要があります。 ``Required[]`` および ``NotRequired[]`` のランタイム実装もこの制限を強制する場合があります。

TypedDict の :ref:`代替機能構文 <typeddict-functional-syntax>` も ``Required[]``、``NotRequired[]``、および ``ReadOnly[]`` をサポートします::

   Movie = TypedDict('Movie', {'name': str, 'year': NotRequired[int]})


``total=False`` との相互作用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``total=False`` で宣言された TypedDict は、すべてのキーが ``NotRequired[]`` としてマークされた暗黙的な ``total=True`` 定義を持つ TypedDict と同等です。

したがって::

   class _MovieBase(TypedDict):  # 暗黙的に total=True
       title: str

   class Movie(_MovieBase, total=False):
       year: int


は次のように同等です::

   class _MovieBase(TypedDict):
       title: str

   class Movie(_MovieBase):
       year: NotRequired[int]


``Annotated[]`` との相互作用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``Required[]`` および ``NotRequired[]`` は ``Annotated[]`` と一緒に使用できます。 任意のネスト順序で::

   class Movie(TypedDict):
       title: str
       year: NotRequired[Annotated[int, ValueRange(-9999, 9999)]]  # ok

::

   class Movie(TypedDict):
       title: str
       year: Annotated[NotRequired[int], ValueRange(-9999, 9999)]  # ok

特に、項目の最外部の注釈として ``Annotated[]`` を許可することで、注釈の非型付け使用との相互運用性が向上します。 これにより、常に ``Annotated[]`` を最外部の注釈として使用することができます (`discussion <https://bugs.python.org/issue46491>`__)。


読み取り専用項目
------------------------------------------------------------------------------------------

(元々は :pep:`705` で指定されていました。)

.. _`readonly`:

``typing.ReadOnly`` 型修飾子
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``typing.ReadOnly`` :term:`type qualifier` は、``TypedDict`` 定義で宣言された項目が変更 (追加、変更、削除) できないことを示すために使用されます::

    from typing import ReadOnly

    class Band(TypedDict):
        name: str
        members: ReadOnly[list[str]]

    blur: Band = {"name": "blur", "members": []}
    blur["name"] = "Blur"  # OK: "name" は読み取り専用ではありません
    blur["members"] = ["Damon Albarn"]  # 型チェックエラー: "members" は読み取り専用です
    blur["members"].append("Damon Albarn")  # OK: リストは変更可能です


他の特殊な型との相互作用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``ReadOnly[]`` は ``Required[]``、``NotRequired[]``、および ``Annotated[]`` と一緒に使用できます。 任意のネスト順序で::

    class Movie(TypedDict):
        title: ReadOnly[Required[str]]  # OK
        year: ReadOnly[NotRequired[Annotated[int, ValueRange(-9999, 9999)]]]  # OK

::

    class Movie(TypedDict):
        title: Required[ReadOnly[str]]  # OK
        year: Annotated[NotRequired[ReadOnly[int]], ValueRange(-9999, 9999)]  # OK


継承
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

サブクラスは読み取り専用項目を非読み取り専用として再宣言し、変更できるようにすることができます::

    class NamedDict(TypedDict):
        name: ReadOnly[str]

    class Album(NamedDict):
        name: str
        year: int

    album: Album = { "name": "Flood", "year": 1990 }
    album["year"] = 1973
    album["name"] = "Dark Side Of The Moon"  # OK: "name" は Album では読み取り専用ではありません

読み取り専用項目が再宣言されていない場合、それは読み取り専用のままです::

    class Album(NamedDict):
        year: int

    album: Album = { "name": "Flood", "year": 1990 }
    album["name"] = "Dark Side Of The Moon"  # 型チェックエラー: "name" は Album では読み取り専用です

サブクラスは読み取り専用項目の値の型を狭めることができます::

    class AlbumCollection(TypedDict):
        albums: ReadOnly[Collection[Album]]

    class RecordShop(AlbumCollection):
        name: str
        albums: ReadOnly[list[Album]]  # OK: "albums" は AlbumCollection では読み取り専用です

サブクラスは、スーパークラスで読み取り専用だが必須ではない項目を必須にすることができます::

    class OptionalName(TypedDict):
        name: ReadOnly[NotRequired[str]]

    class RequiredName(OptionalName):
        name: ReadOnly[Required[str]]

    d: RequiredName = {}  # 型チェックエラー: "name" が必要です

サブクラスはこれらのルールを組み合わせることができます::

    class OptionalIdent(TypedDict):
        ident: ReadOnly[NotRequired[str | int]]

    class User(OptionalIdent):
        ident: str  # 必須、変更可能、および int ではない

これらはすべて :term:`structural` 型付けの結果にすぎませんが、ここで強調されています。 これは、動作が :pep:`589` で指定されたルールと異なるためです。

割り当て可能性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*このセクションは、ReadOnly の導入前に作成された上記の割り当て可能性ルールを更新します*

TypedDict 型 ``B`` は、``B`` が ``A`` に :term:`structurally <structural>` 割り当て可能である場合、TypedDict 型 ``A`` に :term:`assignable` です。 これは、次のすべてが満たされている場合にのみ当てはまります:

* ``A`` の各項目に対して、``B`` は対応するキーを持ちます。 ただし、項目が読み取り専用で、必須ではなく、トップ値型 (``ReadOnly[NotRequired[object]]``) の場合を除きます。
* ``A`` の各項目に対して、``B`` が対応するキーを持つ場合、``B`` の対応する値の型は ``A`` の値の型に割り当て可能です。
* ``A`` の各非読み取り専用項目に対して、その値の型は ``B`` の対応する値の型に割り当て可能であり、対応するキーは ``B`` で読み取り専用ではありません。
* ``A`` の各必須キーに対して、対応するキーは ``B`` で必須です。
* ``A`` の各非必須キーに対して、項目が ``A`` で読み取り専用でない場合、対応するキーは ``B`` で必須ではありません。

議論:

* TypedDict で指定されていないすべての項目は暗黙的に値型 ``ReadOnly[NotRequired[object]]`` を持ちます。

* 読み取り専用項目は変更できないため、共変に動作します。 これは、``Sequence`` などのコンテナ型と同様であり、非読み取り専用項目とは異なります。 例::

    class A(TypedDict):
        x: ReadOnly[int | None]

    class B(TypedDict):
        x: int

    def f(a: A) -> None:
        print(a["x"] or 0)

    b: B = {"x": 1}
    f(b)  # 型チェッカーによって受け入れられます

* 明示的なキー ``'x'`` を持たない TypedDict 型 ``A`` は、非必須キー ``'x'`` を持つ TypedDict 型 ``B`` に :term:`assignable` ではありません。 これは、ランタイムでキー ``'x'`` が存在し、:term:`structural` 型付けによって ``A`` を通じて表示されない :term:`inconsistent <consistent>` 型を持つ可能性があるためです。 このルールの唯一の例外は、項目が読み取り専用であり、値型がトップ型 (``object``) である場合です。 例::

    class A(TypedDict):
        x: int

    class B(TypedDict):
        x: int
        y: ReadOnly[NotRequired[object]]

    a: A = { "x": 1 }
    b: B = a  # 型チェッカーによって受け入れられます

更新メソッド
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

既存の型チェックルールに加えて、型チェッカーは、読み取り専用項目を持つ TypedDict がそのキーを宣言する別の TypedDict で更新された場合にエラーを出すべきです::

    class A(TypedDict):
        x: ReadOnly[int]
        y: int

    a1: A = { "x": 1, "y": 2 }
    a2: A = { "x": 3, "y": 4 }
    a1.update(a2)  # 型チェックエラー: "x" は A で読み取り専用です

宣言された値がボトム型 (:data:`~typing.Never`) でない限り::

    class B(TypedDict):
        x: NotRequired[typing.Never]
        y: ReadOnly[int]

    def update_a(a: A, b: B) -> None:
        a.update(b)  # 型チェッカーによって受け入れられます: "x" は b に設定できません

注: 何も ``Never`` 型に一致しないため、これで注釈された項目は存在しない必要があります。

キーワード引数の型付け
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

セクション :ref:`unpack-kwargs` で説明されているように、アンパックされた ``TypedDict`` は ``**kwargs`` を注釈するために使用できます。 この方法で使用される ``TypedDict`` の 1 つ以上の項目を読み取り専用としてマークしても、メソッドの型シグネチャには影響しません。 ただし、関数の本体で項目が変更されるのを防ぎます::

    class Args(TypedDict):
        key1: int
        key2: str

    class ReadOnlyArgs(TypedDict):
        key1: ReadOnly[int]
        key2: ReadOnly[str]

    class Function(Protocol):
        def __call__(self, **kwargs: Unpack[Args]) -> None: ...

    def impl(**kwargs: Unpack[ReadOnlyArgs]) -> None:
        kwargs["key1"] = 3  # 型チェックエラー: key1 は読み取り専用です

    fn: Function = impl  # 型チェッカーによって受け入れられます: 関数シグネチャは同一です
