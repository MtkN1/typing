.. _`callables`:

呼び出し可能
==========================================================================================

用語
------------------------------------------------------------------------------------------

このセクションおよびこの仕様全体で、「パラメーター」という用語は、関数に渡される引数（または複数の引数）の値を受け取る関数に関連付けられた名前付きシンボルを指します。「引数」という用語は、関数が呼び出されたときに関数に渡される値を指します。

Python は 5 種類のパラメーターをサポートしています：位置のみ、キーワードのみ、標準（位置またはキーワード）、可変位置（``*args``）、および可変キーワード（``**kwargs``）。位置のみのパラメーターは位置引数のみを受け入れることができ、キーワードのみのパラメーターはキーワード引数のみを受け入れることができます。標準パラメーターは位置引数またはキーワード引数のいずれかを受け入れることができます。``*args`` および ``**kwargs`` 形式のパラメーターは可変であり、それぞれゼロまたは複数の位置引数またはキーワード引数を受け入れます。

以下の例では、``a`` は位置のみのパラメーター、``b`` は標準（位置またはキーワード）パラメーター、``c`` はキーワードのみのパラメーター、``args`` は追加の位置引数を受け入れる可変パラメーター、``kwargs`` は追加のキーワード引数を受け入れる可変パラメーターです::

    def func(a: str, /, b, *args, c=0, **kwargs) -> None:
        ...

関数の「シグネチャ」は、パラメーターのリスト（名前、種類、オプションの宣言された型、およびデフォルトの引数値があるかどうかを含む）とその戻り値の型を指します。上記の関数のシグネチャは ``(a: str, /, b, *args, c=..., **kwargs) -> None`` です。ここで、パラメーター ``c`` のデフォルト引数値は ``...`` として示されています。これは、デフォルト値の存在がシグネチャの一部と見なされますが、特定の値は含まれないためです。

「入力シグネチャ」という用語は、関数のパラメーターのみを指すために使用されます。上記の例では、入力シグネチャは ``(a: str, /, b, *args, c=..., **kwargs)`` です。

位置のみのパラメーター
------------------------------------------------------------------------------------------

関数シグネチャ内では、位置のみのパラメーターは非位置のみのパラメーターからスラッシュ（'/'）で区切られます。このスラッシュはパラメーターを表すのではなく、区切り文字として機能します。この例では、``a`` は位置のみのパラメーターであり、``b`` は標準（位置またはキーワード）パラメーターです::

    def func(a: int, /, b: int) -> None:
        ...

    func(1, 2)  # OK
    func(1, b=2)  # OK
    func(a=1, b=2)  # エラー

``/`` 区切り文字のサポートは Python 3.8 で導入されました (:pep:`570`)。Python の以前のバージョンとの互換性のために、タイプシステムは :ref:`二重の先頭アンダースコア <pos-only-double-underscore>` を使用して位置のみのパラメーターを指定することもサポートしています。

デフォルト引数値
------------------------------------------------------------------------------------------

特定のケースでは、パラメーターのデフォルト引数値を省略することが望ましい場合があります。例としては、スタブファイル内の関数定義やプロトコルまたは抽象基底クラス内のメソッドなどがあります。そのような場合、デフォルト値は省略記号として与えることができます。例えば::

  def func(x: AnyStr, y: AnyStr = ...) -> AnyStr: ...

非省略記号のデフォルト値が存在し、その型が静的に評価できる場合、型チェッカーはこの型が宣言されたパラメーターの型に :term:`割り当て可能 <assignable>` であることを確認する必要があります::

    def func(x: int = 0): ...  # OK
    def func(x: int | None = None): ...  # OK
    def func(x: int = 0.0): ...  # エラー
    def func(x: int = None): ...  # エラー

.. _`annotating-args-kwargs`:

``*args`` および ``**kwargs`` の注釈
------------------------------------------------------------------------------------------

ランタイム時には、可変位置パラメーター（``*args``）の型は ``tuple`` であり、可変キーワードパラメーター（``**kwargs``）の型は ``dict`` です。ただし、これらのパラメーターに注釈を付ける場合、型注釈は ``tuple`` または ``dict`` 内の項目の型を指します（``Unpack`` が使用されない限り）。

したがって、次の定義は::

  def func(*args: str, **kwargs: int): ...

この関数は任意の数の型 ``str`` の位置引数と任意の数の型 ``int`` のキーワード引数を受け入れることを意味します。例えば、次のすべては有効な引数を持つ関数呼び出しを表します::

  func('a', 'b', 'c')
  func(x=1, y=2)
  func('', z=0)

関数 ``func`` の本体では、パラメーター ``args`` の型は ``tuple[str, ...]`` であり、パラメーター ``kwargs`` の型は ``dict[str, int]`` です。

.. _unpack-kwargs:

キーワード引数のための ``Unpack``
------------------------------------------------------------------------------------------

``typing.Unpack`` にはタイプシステムで 2 つの使用例があります：

* :pep:`646` によって導入された、可変ジェネリックを含む特定の操作の後方互換形式。詳細は ``TypeVarTuple`` のセクションを参照してください。
* :pep:`692` によって導入された、関数の ``**kwargs`` を注釈する方法。

この第 2 の使用法はこのセクションで説明されています。次の例は::

    from typing import TypedDict, Unpack

    class Movie(TypedDict):
        name: str
        year: int

    def foo(**kwargs: Unpack[Movie]) -> None: ...

この ``**kwargs`` が ``Movie`` によって指定された 2 つのキーワード引数（すなわち、型 ``str`` の ``name`` キーワードと型 ``int`` の ``year`` キーワード）で構成されていることを意味します。これは関数が次のように呼び出されるべきであることを示しています::

    kwargs: Movie = {"name": "Life of Brian", "year": 1979}

    foo(**kwargs)                               # OK!
    foo(name="The Meaning of Life", year=1983)  # OK!

``Unpack`` が使用される場合、型チェッカーは関数本体内の ``kwargs`` を ``TypedDict`` として扱います::

    def foo(**kwargs: Unpack[Movie]) -> None:
        assert_type(kwargs, Movie)  # OK!


標準辞書を使用した関数呼び出し
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

関数に ``**kwargs`` 引数として型 ``dict[str, object]`` の辞書を渡す場合、関数の ``**kwargs`` が ``Unpack`` で注釈されている場合、型チェッカーエラーを生成する必要があります。一方、標準の型指定されていない辞書を使用する関数の動作は型チェッカーによって異なる場合があります。例えば::

    def func(**kwargs: Unpack[Movie]) -> None: ...

    movie: dict[str, object] = {"name": "Life of Brian", "year": 1979}
    func(**movie)  # 間違い! Movie は型 dict[str, object] です

    typed_movie: Movie = {"name": "The Meaning of Life", "year": 1983}
    func(**typed_movie)  # OK!

    another_movie = {"name": "Life of Brian", "year": 1979}
    func(**another_movie)  # 型チェッカーによります。

キーワードの衝突
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``TypedDict`` を使用して ``**kwargs`` を型付けする場合、その ``TypedDict`` が関数のシグネチャですでに定義されているキーを含む可能性があります。重複した名前が標準パラメーターである場合、型チェッカーによってエラーが報告されるべきです。重複した名前が位置のみのパラメーターである場合、エラーは生成されるべきではありません。例えば::

    def foo(name, **kwargs: Unpack[Movie]) -> None: ...     # 間違い! "name" は常に最初のパラメーターにバインドされます。

    def foo(name, /, **kwargs: Unpack[Movie]) -> None: ...  # OK! "name" は位置のみのパラメーターであるため、**kwargs は "name" キーワードを含むことができます。

必須および非必須のキー
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

デフォルトでは、``TypedDict`` 内のすべてのキーは必須です。この動作は辞書の ``total`` パラメーターを ``False`` に設定することで上書きできます。さらに、:pep:`655` は新しい型修飾子 - ``typing.Required`` および ``typing.NotRequired`` - を導入し、特定のキーが必須かどうかを指定できるようにしました::

    class Movie(TypedDict):
        title: str
        year: NotRequired[int]

``TypedDict`` を使用して ``**kwargs`` を型付けする場合、すべての必須および非必須のキーは、必須および非必須の関数キーワードパラメーターに対応する必要があります。したがって、必須のキーが呼び出し元でサポートされていない場合、型チェッカーによってエラーが報告されるべきです。

割り当て
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``**kwargs: Unpack[Movie]`` で型付けされた関数の割り当てと別の呼び出し可能な型の割り当ては、以下に記載されたシナリオに対してのみ型チェックを通過するべきです。

ソースと宛先の両方が ``**kwargs`` を含む
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

宛先関数とソース関数の両方が ``**kwargs: Unpack[TypedDict]`` パラメーターを持ち、宛先関数の ``TypedDict`` がソース関数の ``TypedDict`` に :term:`割り当て可能 <assignable>` であり、残りのパラメーターが割り当て可能である場合::

    class Animal(TypedDict):
        name: str

    class Dog(Animal):
        breed: str

    def accept_animal(**kwargs: Unpack[Animal]): ...
    def accept_dog(**kwargs: Unpack[Dog]): ...

    accept_dog = accept_animal  # OK! Dog 型の式を Animal 型の変数に割り当てることができます。

    accept_animal = accept_dog  # 間違い! Animal 型の式を Dog 型の変数に割り当てることはできません。

.. _PEP 692 assignment dest no kwargs:

ソースが ``**kwargs`` を含み、宛先が含まない
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

宛先の呼び出し可能な関数が ``**kwargs`` を含まず、ソースの呼び出し可能な関数が ``**kwargs: Unpack[TypedDict]`` を含み、宛先関数のキーワード引数がソース関数の ``TypedDict`` の対応するキーに :term:`割り当て可能 <assignable>` である場合。さらに、必須ではないキーはオプションの関数引数に対応し、必須キーは必須の関数引数に対応する必要があります。再び、残りのパラメーターは割り当て可能でなければなりません。前の例を続けます::

    class Example(TypedDict):
        animal: Animal
        string: str
        number: NotRequired[int]

    def src(**kwargs: Unpack[Example]): ...
    def dest(*, animal: Dog, string: str, number: int = ...): ...

    dest = src  # OK!

宛先関数のパラメーターが ``TypedDict`` のキーと値に割り当て可能である必要があるため、これらのパラメーターはキーワードのみでなければなりません::

    def dest(dog: Dog, string: str, number: int = ...): ...

    dog: Dog = {"name": "Daisy", "breed": "labrador"}

    dest(dog, "some string")  # OK!

    dest = src                # 型チェッカーエラー!
    dest(dog, "some string")  # 同じ呼び出しがランタイムで失敗します。src はキーワード引数を期待しているためです。

宛先の呼び出し可能な関数が ``**kwargs: Unpack[TypedDict]`` を含み、ソースの呼び出し可能な関数が ``**kwargs`` を含まない場合は許可されるべきではありません。これは、追加のキーワード引数が渡されていないことを確認できないためです。サブクラスのインスタンスが基本クラス型の変数に割り当てられ、その後、宛先の呼び出し可能な関数の呼び出しで展開される場合::

    def dest(**kwargs: Unpack[Animal]): ...
    def src(name: str): ...

    dog: Dog = {"name": "Daisy", "breed": "Labrador"}
    animal: Animal = dog

    dest = src      # 間違い!
    dest(**animal)  # ランタイムで失敗します。

同様の状況は、継承なしでも発生する可能性があります。``TypedDict`` 間の :term:`割り当て可能性 <assignable>` は :term:`構造的 <structural>` であるためです。

ソースが型指定されていない ``**kwargs`` を含む
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

宛先の呼び出し可能な関数が ``**kwargs: Unpack[TypedDict]`` を含み、ソースの呼び出し可能な関数が型指定されていない ``**kwargs`` を含む場合::

    def src(**kwargs): ...
    def dest(**kwargs: Unpack[Movie]): ...

    dest = src  # OK!

ソースが従来の方法で型指定された ``**kwargs: T`` を含む
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

宛先の呼び出し可能な関数が ``**kwargs: Unpack[TypedDict]`` を含み、ソースの呼び出し可能な関数が従来の方法で型指定された ``**kwargs: T`` を含み、宛先関数の ``TypedDict`` の各フィールドが型 ``T`` の変数に :term:`割り当て可能 <assignable>` である場合::

    class Vehicle:
        ...

    class Car(Vehicle):
        ...

    class Motorcycle(Vehicle):
        ...

    class Vehicles(TypedDict):
        car: Car
        moto: Motorcycle

    def dest(**kwargs: Unpack[Vehicles]): ...
    def src(**kwargs: Vehicle): ...

    dest = src  # OK!

一方、宛先の呼び出し可能な関数が型指定されていないまたは従来の方法で型指定された ``**kwargs: T`` を含み、ソースの呼び出し可能な関数が ``**kwargs: Unpack[TypedDict]`` で型指定されている場合、エラーが生成されるべきです。これは、従来の方法で型指定された ``**kwargs`` がキーワード名のチェックを行わないためです。

要約すると、関数パラメーターは逆変の動作をし、関数の戻り値の型は共変の動作をするべきです。

関数内で他の関数に kwargs を渡す
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:ref:`前のポイント <PEP 692 assignment dest no kwargs>` は、サブクラスのインスタンスを基本クラス型の変数に割り当て、その後、宛先の呼び出し可能な関数の呼び出しで展開される場合、追加のキーワード引数を渡す可能性の問題に言及しています。次の例を考えてみましょう::

    class Animal(TypedDict):
        name: str

    class Dog(Animal):
        breed: str

    def takes_name(name: str): ...

    dog: Dog = {"name": "Daisy", "breed": "Labrador"}
    animal: Animal = dog

    def foo(**kwargs: Unpack[Animal]):
        print(kwargs["name"].capitalize())

    def bar(**kwargs: Unpack[Animal]):
        takes_name(**kwargs)

    def baz(animal: Animal):
        takes_name(**animal)

    def spam(**kwargs: Unpack[Animal]):
        baz(kwargs)

    foo(**animal)   # OK! foo は 'Animal' のキーワードのみを期待し、使用します。

    bar(**animal)   # 間違い! これはランタイムで失敗します。'breed' キーワードが 'takes_name' にも渡されるためです。

    spam(**animal)  # 間違い! 再び、'breed' キーワードが最終的に 'takes_name' に渡されます。

上記の例では、``foo`` への呼び出しはランタイムで問題を引き起こしません。たとえ ``foo`` が ``Animal`` 型の ``kwargs`` を期待していても、追加の引数を受け取っても問題ありません。必要なものだけを読み取り、使用し、追加の値を完全に無視します。

``bar`` および ``spam`` への呼び出しは失敗します。予期しないキーワード引数が ``takes_name`` 関数に渡されるためです。

したがって、アンパックされた ``TypedDict`` でヒントされた ``kwargs`` は、アンパックされたキーワード引数が関数のシグネチャにも含まれている場合にのみ他の関数に渡すことができます。そうでない場合、型チェッカーはエラーを生成するべきです。

上記の ``bar`` 関数に似たケースでは、必要なフィールドを明示的に参照解除し、それらを引数として使用して関数呼び出しを行うことで問題を回避できます::

    def bar(**kwargs: Unpack[Animal]):
        name = kwargs["name"]
        takes_name(name)

``TypedDict`` 以外の型で ``Unpack`` を使用する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``TypedDict`` は ``**kwargs`` を型付けするための唯一の許可された異種型です。したがって、``**kwargs`` を型付けする文脈では、``TypedDict`` 以外の型で ``Unpack`` を使用することは許可されず、そのような場合には型チェッカーがエラーを生成するべきです。

.. _`callable`:

呼び出し可能
------------------------------------------------------------------------------------------

``Callable`` 特殊形式は、型式内で関数のシグネチャを指定するために使用できます。構文は ``Callable[[Param1Type, Param2Type], ReturnType]`` です。例えば::

    from collections.abc import Callable

    def func(cb: Callable[[int], str]) -> None:
        ...

    x: Callable[[], str]

``Callable`` を使用して指定されたパラメーターは位置のみと見なされます。``Callable`` 形式はキーワードのみのパラメーター、可変パラメーター、またはデフォルト引数値を指定する方法を提供しません。これらの使用例については、`コールバックプロトコル`_ のセクションを参照してください。

``Callable`` における ``...`` の意味
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``Callable`` 特殊形式は、パラメーター型のリストの代わりに ``...`` を使用することをサポートしています。これは、型が任意の入力シグネチャと一致することを示す :term:`漸進的形式 <consistent>` です::

    cb1: Callable[..., str]
    cb1 = lambda x: str(x)  # OK
    cb1 = lambda : ""  # OK

    cb2: Callable[[], str] = cb1  # OK

``...`` は ``Concatenate`` と共に使用することもできます。この場合、``...`` の前のパラメーターは入力シグネチャに存在し、割り当て可能である必要がありますが、追加のパラメーターは許可されます::

    cb3: Callable[Concatenate[int, ...], str]
    cb3 = lambda x: str(x)  # OK
    cb3 = lambda a, b, c: str(a)  # OK
    cb3 = lambda : ""  # エラー
    cb3 = lambda *, a: str(a)  # エラー


関数定義の入力シグネチャに ``*args`` および ``**kwargs`` パラメーターが含まれ、両方が ``Any`` として型付けされている場合（明示的にまたは注釈がないために暗黙的に）、型チェッカーはこれを ``...`` と同等と見なすべきです。シグネチャ内の他のパラメーターは影響を受けず、シグネチャの一部として保持されます::

    class Proto1(Protocol):
        def __call__(self, *args: Any, **kwargs: Any) -> None: ...

    class Proto2(Protocol):
        def __call__(self, a: int, /, *args, **kwargs) -> None: ...

    class Proto3(Protocol):
        def __call__(self, a: int, *args: Any, **kwargs: Any) -> None: ...

    class Proto4[**P](Protocol):
        def __call__(self, a: int, *args: P.args, **kwargs: P.kwargs) -> None: ...

    def func(p1: Proto1, p2: Proto2, p3: Proto3):
        assert_type(p1, Callable[..., None])  # OK
        assert_type(p2, Callable[Concatenate[int, ...], None])  # OK
        assert_type(p3, Callable[..., None])  # エラー
        assert_type(p3, Proto4[...])  # OK

    class A:
        def method(self, a: int, /, *args: Any, k: str, **kwargs: Any) -> None:
            pass

    class B(A):
        # このオーバーライドは、親のメソッドに割り当て可能であるため OK です。
        def method(self, a: float, /, b: int, *, k: str, m: str) -> None:
            pass


``...`` 構文は、ジェネリッククラスまたは型エイリアスで :ref:`ParamSpec の特殊化値 <paramspec_valid_use_locations>` を提供するためにも使用できます。例えば::

    type Callback[**P] = Callable[P, str]

    def func(cb: Callable[[], str]) -> None:
        f: Callback[...] = cb  # OK

``...`` がシグネチャ連結と共に使用される場合、``...`` 部分は引き続き任意の入力パラメーターと一致します::

    type CallbackWithInt[**P] = Callable[Concatenate[int, P], str]
    type CallbackWithStr[**P] = Callable[Concatenate[str, P], str]

    def func(cb: Callable[[int, str], str]) -> None:
        f1: Callable[Concatenate[int, ...], str] = cb # OK
        f2: Callable[Concatenate[str, ...], str] = cb # エラー
        f3: CallbackWithInt[...] = cb  # OK
        f4: CallbackWithStr[...] = cb  # エラー

.. _`callback-protocols`:

コールバックプロトコル
------------------------------------------------------------------------------------------

プロトコルを使用して、``Callable`` 特殊形式として表現することが不可能な柔軟なコールバック型を定義できます。これには、キーワードパラメーター、可変パラメーター、デフォルト引数値、およびオーバーロードが含まれます。プロトコルとして ``__call__`` メンバーを持つプロトコルとして定義できます::

  from typing import Protocol

  class Combiner(Protocol):
      def __call__(self, *args: bytes, max_len: int | None = None) -> list[bytes]: ...

  def good_cb(*args: bytes, max_len: int | None = None) -> list[bytes]:
      ...
  def bad_cb(*args: bytes, max_items: int | None) -> list[bytes]:
      ...

  comb: Combiner = good_cb  # OK
  comb = bad_cb  # エラー! 引数 2 はコールバック内の異なるパラメーター名と種類のため割り当て可能ではありません

コールバックプロトコルと ``Callable[...]`` 型は一般的に相互に交換可能です。


呼び出し可能な型の割り当てルール
------------------------------------------------------------------------------------------

呼び出し可能な型 ``B`` は、``B`` の戻り値の型が ``A`` の戻り値の型に割り当て可能であり、``B`` の入力シグネチャが ``A`` の入力シグネチャが受け入れるすべての組み合わせの引数を受け入れる場合、呼び出し可能な型 ``A`` に :term:`割り当て可能 <assignable>` です。以下に説明するすべての特定の割り当てルールは、この一般的なルールから派生しています。


パラメーターの型
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

呼び出し可能な型は、その戻り値の型に関して共変であり、パラメーターの型に関して逆変です。これは、呼び出し可能な ``B`` が ``A`` に :term:`割り当て可能 <assignable>` であるためには、``A`` のパラメーターの型が ``B`` のパラメーターに割り当て可能である必要があることを意味します。例えば、``(x: float) -> int`` は ``(x: int) -> float`` に割り当て可能です::

    def func(cb: Callable[[float], int]):
        f1: Callable[[int], float] = cb  # OK


パラメーターの種類
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

呼び出し可能な ``B`` は、``A`` のすべてのキーワードのみのパラメーターが ``B`` にキーワードのみのパラメーターまたは標準（位置またはキーワード）パラメーターとして存在する場合にのみ ``A`` に :term:`割り当て可能 <assignable>` です。例えば、``(a: int) -> None`` は ``(*, a: int) -> None`` に割り当て可能ですが、逆は真ではありません。キーワードのみのパラメーターの順序は割り当て可能性の目的で無視されます::

    class KwOnly(Protocol):
        def __call__(self, *, b: int, a: int) -> None: ...

    class Standard(Protocol):
        def __call__(self, a: int, b: int) -> None: ...

    def func(standard: Standard, kw_only: KwOnly):
        f1: KwOnly = standard  # OK
        f2: Standard = kw_only  # エラー

同様に、呼び出し可能な ``B`` は、``A`` のすべての位置のみのパラメーターが ``B`` に位置のみのパラメーターまたは標準（位置またはキーワード）パラメーターとして存在する場合にのみ ``A`` に割り当て可能です。位置のみのパラメーターの名前は割り当て可能性の目的で無視されます::

    class PosOnly(Protocol):
        def __call__(self, not_a: int, /) -> None: ...

    class Standard(Protocol):
        def __call__(self, a: int) -> None: ...

    def func(standard: Standard, pos_only: PosOnly):
        f1: PosOnly = standard  # OK
        f2: Standard = pos_only  # エラー


``*args`` パラメーター
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

呼び出し可能な ``A`` が ``*args`` パラメーターを持つシグネチャを持つ場合、呼び出し可能な ``B`` も ``*args`` パラメーターを持つ必要があり、``A`` の ``*args`` パラメーターの型が ``B`` の ``*args`` パラメーターに割り当て可能である必要があります::

    class NoArgs(Protocol):
        def __call__(self) -> None: ...

    class IntArgs(Protocol):
        def __call__(self, *args: int) -> None: ...

    class FloatArgs(Protocol):
        def __call__(self, *args: float) -> None: ...

    def func(no_args: NoArgs, int_args: IntArgs, float_args: FloatArgs):
        f1: NoArgs = int_args  # OK
        f2: NoArgs = float_args  # OK

        f3: IntArgs = no_args  # エラー: *args パラメーターがありません
        f4: IntArgs = float_args  # OK

        f5: FloatArgs = no_args  # エラー: *args パラメーターがありません
        f6: FloatArgs = int_args  # エラー: float は int に割り当て可能ではありません

呼び出し可能な ``A`` が 1 つ以上の位置のみのパラメーターを持つシグネチャを持つ場合、呼び出し可能な ``B`` は ``A`` に割り当て可能です。``B`` が ``*args`` パラメーターを持ち、``A`` の他の一致しない位置のみのパラメーターの型が ``B`` の ``*args`` パラメーターに割り当て可能である場合::

    class PosOnly(Protocol):
        def __call__(self, a: int, b: str, /) -> None: ...

    class IntArgs(Protocol):
        def __call__(self, *args: int) -> None: ...

    class IntStrArgs(Protocol):
        def __call__(self, *args: int | str) -> None: ...

    class StrArgs(Protocol):
        def __call__(self, a: int, /, *args: str) -> None: ...

    class Standard(Protocol):
        def __call__(self, a: int, b: str) -> None: ...

    def func(int_args: IntArgs, int_str_args: IntStrArgs, str_args: StrArgs):
        f1: PosOnly = int_args  # エラー: str は int に割り当て可能ではありません
        f2: PosOnly = int_str_args  # OK
        f3: PosOnly = str_args  # OK
        f4: IntStrArgs = str_args  # エラー: int | str は str に割り当て可能ではありません
        f5: IntStrArgs = int_args  # エラー: int | str は int に割り当て可能ではありません
        f6: StrArgs = int_str_args  # OK
        f7: StrArgs = int_args  # エラー: str は int に割り当て可能ではありません
        f8: IntArgs = int_str_args  # OK
        f9: IntArgs = str_args  # エラー: int は str に割り当て可能ではありません
        f10: Standard = int_str_args  # エラー: キーワードパラメーター a および b が不足しています
        f11: Standard = str_args  # エラー: キーワードパラメーター b が不足しています


``**kwargs`` パラメーター
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

呼び出し可能な ``A`` が ``**kwargs`` パラメーターを持つシグネチャを持つ場合（アンパックされた ``TypedDict`` 型注釈がない場合）、呼び出し可能な ``B`` も ``**kwargs`` パラメーターを持つ必要があり、``A`` の ``**kwargs`` パラメーターの型が ``B`` の ``**kwargs`` パラメーターに割り当て可能である必要があります::

    class NoKwargs(Protocol):
        def __call__(self) -> None: ...

    class IntKwargs(Protocol):
        def __call__(self, **kwargs: int) -> None: ...

    class FloatKwargs(Protocol):
        def __call__(self, **kwargs: float) -> None: ...

    def func(no_kwargs: NoKwargs, int_kwargs: IntKwargs, float_kwargs: FloatKwargs):
        f1: NoKwargs = int_kwargs  # OK
        f2: NoKwargs = float_kwargs  # OK

        f3: IntKwargs = no_kwargs  # エラー: **kwargs パラメーターがありません
        f4: IntKwargs = float_kwargs  # OK

        f5: FloatKwargs = no_kwargs  # エラー: **kwargs パラメーターがありません
        f6: FloatKwargs = int_kwargs  # エラー: float は int に割り当て可能ではありません

呼び出し可能な ``A`` が 1 つ以上のキーワードのみのパラメーターを持つシグネチャを持つ場合、呼び出し可能な ``B`` は ``A`` に割り当て可能です。``B`` が ``**kwargs`` パラメーターを持ち、``A`` の他の一致しないキーワードのみのパラメーターの型が ``B`` の ``**kwargs`` パラメーターに割り当て可能である場合::

    class KwOnly(Protocol):
        def __call__(self, *, a: int, b: str) -> None: ...

    class IntKwargs(Protocol):
        def __call__(self, **kwargs: int) -> None: ...

    class IntStrKwargs(Protocol):
        def __call__(self, **kwargs: int | str) -> None: ...

    class StrKwargs(Protocol):
        def __call__(self, *, a: int, **kwargs: str) -> None: ...

    class Standard(Protocol):
        def __call__(self, a: int, b: str) -> None: ...

    def func(int_kwargs: IntKwargs, int_str_kwargs: IntStrKwargs, str_kwargs: StrKwargs):
        f1: KwOnly = int_kwargs  # エラー: str は int に割り当て可能ではありません
        f2: KwOnly = int_str_kwargs  # OK
        f3: KwOnly = str_kwargs  # OK
        f4: IntStrKwargs = str_kwargs  # エラー: int | str は str に割り当て可能ではありません
        f5: IntStrKwargs = int_kwargs  # エラー: int | str は int に割り当て可能ではありません
        f6: StrKwargs = int_str_kwargs  # OK
        f7: StrKwargs = int_kwargs  # エラー: str は int に割り当て可能ではありません
        f8: IntKwargs = int_str_kwargs  # OK
        f9: IntKwargs = str_kwargs  # エラー: int は str に割り当て可能ではありません
        f10: Standard = int_str_kwargs  # エラー: 位置引数を受け入れません
        f11: Standard = str_kwargs  # エラー: 位置引数を受け入れません

アンパックされた ``TypedDict`` を持つ ``**kwargs`` を含む呼び出し可能なシグネチャの割り当てルールは、上記のセクション :ref:`unpack-kwargs` に記載されています。


ParamSpecs を含むシグネチャ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``*args: P.args, **kwargs: P.kwargs`` を含むシグネチャは、``P`` でパラメーター化された ``Callable`` と同等です::

    class ProtocolWithP[**P](Protocol):
        def __call__(self, *args: P.args, **kwargs: P.kwargs) -> None: ...

    type TypeAliasWithP[**P] = Callable[P, None]

    def func[**P](proto: ProtocolWithP[P], ta: TypeAliasWithP[P]):
        # これらの 2 つの型は同等です
        f1: TypeAliasWithP[P] = proto  # OK
        f2: ProtocolWithP[P] = ta  # OK


デフォルト引数値
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

呼び出し可能な ``C`` がデフォルト引数値を持つパラメーター ``x`` を持ち、``A`` が ``C`` と同じであり、``x`` にデフォルト引数がない場合、``C`` は ``A`` に :term:`割り当て可能 <assignable>` です。また、``C`` が ``A`` と同じであり、パラメーター ``x`` が削除された場合も ``C`` は ``A`` に割り当て可能です::

    class DefaultArg(Protocol):
        def __call__(self, x: int = 0) -> None: ...

    class NoDefaultArg(Protocol):
        def __call__(self, x: int) -> None: ...

    class NoX(Protocol):
        def __call__(self) -> None: ...

    def func(default_arg: DefaultArg):
        f1: NoDefaultArg = default_arg  # OK
        f2: NoX = default_arg  # OK


オーバーロード
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

呼び出し可能な ``B`` が 2 つ以上のシグネチャでオーバーロードされている場合、``B`` のオーバーロードされたシグネチャの少なくとも 1 つが ``A`` に割り当て可能である場合、``B`` は呼び出し可能な ``A`` に :term:`割り当て可能 <assignable>` です::

    class Overloaded(Protocol):
        @overload
        def __call__(self, x: int) -> int: ...
        @overload
        def __call__(self, x: str) -> str: ...

    class IntArg(Protocol):
        def __call__(self, x: int) -> int: ...

    class StrArg(Protocol):
        def __call__(self, x: str) -> str: ...

    class FloatArg(Protocol):
        def __call__(self, x: float) -> float: ...

    def func(overloaded: Overloaded):
        f1: IntArg = overloaded  # OK
        f2: StrArg = overloaded  # OK
        f3: FloatArg = overloaded  # エラー

呼び出し可能な ``A`` が 2 つ以上のシグネチャでオーバーロードされている場合、呼び出し可能な ``B`` は ``A`` に割り当て可能です。``B`` が ``A`` のすべてのシグネチャに割り当て可能である場合::

    class Overloaded(Protocol):
        @overload
        def __call__(self, x: int, y: str) -> float: ...
        @overload
        def __call__(self, x: str) -> complex: ...

    class StrArg(Protocol):
        def __call__(self, x: str) -> complex: ...

    class IntStrArg(Protocol):
        def __call__(self, x: int | str, y: str = "") -> int: ...

    def func(int_str_arg: IntStrArg, str_arg: StrArg):
        f1: Overloaded = int_str_arg  # OK
        f2: Overloaded = str_arg  # エラー
