ジェネリクス
==========================================================================================

Python コードで ``list[str]`` や ``dict[str, int]`` のような型ヒントを見たことがあるかもしれません。 これらの型は、他の型によってパラメータ化されているという点で興味深いです。 ``list[str]`` は単なるリストではなく、文字列のリストです。 このような型パラメータを持つ型は *ジェネリック型* と呼ばれます。

組み込み型 (``list[X]`` など) と同様に、型パラメータを取る独自のジェネリック クラスを定義できます。 このようなユーザー定義のジェネリックは中程度に高度な機能であり、それを使用せずに進むことができます。

.. _generic-classes:

ジェネリック クラスの定義
******************************************************************************************

次に、スタックを表す非常に単純なジェネリック クラスを示します。

.. code-block:: python

   from typing import TypeVar, Generic

   T = TypeVar('T')

   class Stack(Generic[T]):
       def __init__(self) -> None:
           # T 型のアイテムを持つ空のリストを作成します
           self.items: list[T] = []

       def push(self, item: T) -> None:
           self.items.append(item)

       def pop(self) -> T:
           return self.items.pop()

       def empty(self) -> bool:
           return not self.items

``Stack`` クラスは、任意の型のスタックを表すために使用できます: ``Stack[int]``, ``Stack[tuple[int, str]]`` など。

``Stack`` の使用は、``list`` などの組み込みコンテナー型と似ています。

.. code-block:: python

   # 空の Stack[int] インスタンスを構築します
   stack = Stack[int]()
   stack.push(2)
   stack.pop() + 1
   stack.push('x')  # エラー: "Stack" の "push" への引数 1 の型が一致しません。予期される型は "int" です

ジェネリック クラスのインスタンスを作成する場合、型引数は通常推論できます。 型引数を明示的に指定する場合、インスタンスの構築は対応する型チェックが行われます。

.. code-block:: python

   class Box(Generic[T]):
       def __init__(self, content: T) -> None:
           self.content = content

   Box(1)       # OK、推論された型は Box[int] です
   Box[int](1)  # これも OK
   Box[int]('some string')  # エラー: "Box" の引数 1 の型が一致しません。予期される型は "int" です

.. _generic-subclasses:

ジェネリック クラスのサブクラスの定義
******************************************************************************************

ユーザー定義のジェネリック クラスおよび :py:mod:`typing` で定義されたジェネリック クラスは、別のクラス (ジェネリックまたは非ジェネリック) の基本クラスとして使用できます。 例えば:

.. code-block:: python

    from typing import Generic, TypeVar, Mapping, Iterator

    KT = TypeVar('KT')
    VT = TypeVar('VT')

    # これは Mapping のジェネリック サブクラスです
    class MyMap(Mapping[KT, VT]):
        def __getitem__(self, k: KT) -> VT: ...
        def __iter__(self) -> Iterator[KT]: ...
        def __len__(self) -> int: ...

    items: MyMap[str, int]  # OK

    # これは dict の非ジェネリック サブクラスです
    class StrDict(dict[str, str]):
        def __str__(self) -> str:
            return f'StrDict({super().__str__()})'


    data: StrDict[int, int]  # エラー: "StrDict" は型引数を受け取りませんが、2 つ指定されています
    data2: StrDict  # OK

   # これはユーザー定義のジェネリック クラスです
   class Receiver(Generic[T]):
       def accept(self, value: T) -> None: ...

   # これは Receiver のジェネリック サブクラスです
   class AdvancedReceiver(Receiver[T]): ...

.. note::

    クラスがマッピングまたはシーケンスと見なされるには、:py:class:`~typing.Mapping` および :py:class:`~typing.Sequence` から明示的に継承する必要があることに注意してください。 これは、これらのクラスが名目上の型付けであり、:py:class:`~typing.Iterable` のようなプロトコルとは異なり、:ref:`構造的サブタイピング <protocol-types>` を使用するためです。

:py:class:`Generic <typing.Generic>` は、型変数を含む他の基本クラスがある場合、基本クラスから省略できます。 ``Generic[...]`` を基本クラスに含める場合は、他の基本クラスに存在するすべての型変数 (または必要に応じてそれ以上) をリストする必要があります。 型変数の順序は次のルールによって定義されます。

* ``Generic[...]`` が存在する場合、変数の順序は常に ``Generic[...]`` 内の順序によって決定されます。
* 基本クラスに ``Generic[...]`` がない場合、すべての型変数は辞書順 (つまり、最初に出現する順序) で収集されます。

例えば:

.. code-block:: python

   from typing import Generic, TypeVar, Any

   T = TypeVar('T')
   S = TypeVar('S')
   U = TypeVar('U')

   class One(Generic[T]): ...
   class Another(Generic[T]): ...

   class First(One[T], Another[S]): ...
   class Second(One[T], Another[S], Generic[S, U, T]): ...

   x: First[int, str]        # ここで T は int にバインドされ、S は str にバインドされます
   y: Second[int, str, Any]  # ここで T は Any、S は int、U は str です

.. _generic-functions:

ジェネリック関数
******************************************************************************************

型変数を使用してジェネリック関数を定義できます。 これらは、引数または戻り値の型に関係がある関数です。

.. code-block:: python

   from typing import TypeVar, Sequence

   T = TypeVar('T')

   # ジェネリック関数!
   def first(seq: Sequence[T]) -> T:
       return seq[0]

ジェネリック クラスと同様に、型変数は任意の型に置き換えることができます。 つまり、``first`` は任意のシーケンス型で使用でき、戻り値の型はシーケンス アイテムの型から派生します。 例えば:

.. code-block:: python

   reveal_type(first([1, 2, 3]))   # 明らかにされた型は "builtins.int" です
   reveal_type(first(['a', 'b']))  # 明らかにされた型は "builtins.str" です

型変数は 2 つ以上の型の関係を説明するためのものであるため、関数シグネチャに型変数が 1 回しか表示されない場合は通常役に立ちません。

便利なことに、単一の型変数シンボル (上記の ``T`` など) は、複数のジェネリック関数またはクラスで使用できますが、論理的なスコープは各ジェネリック関数またはクラスで異なります。 次の例では、2 つのジェネリック関数で同じ型変数シンボルを再利用します。これらの 2 つの関数は互いに型の関係を共有しません。

.. code-block:: python

   from typing import TypeVar, Sequence

   T = TypeVar('T')

   def first(seq: Sequence[T]) -> T:
       return seq[0]

   def last(seq: Sequence[T]) -> T:
       return seq[-1]

変数は、型変数が含まれるジェネリック クラス、ジェネリック関数、またはジェネリックエイリアスによってバインドされていない限り、その型に型変数を持つべきではありません。

.. _generic-methods-and-generic-self:

ジェネリック メソッドとジェネリック self
******************************************************************************************

型変数をクラス定義でバインドされた型変数とは異なるメソッド シグネチャに使用するだけで、ジェネリック メソッドを定義することもできます。

.. code-block:: python

    # T はこのクラスによってバインドされた型変数です
    class PairedBox(Generic[T]):
        def __init__(self, content: T) -> None:
            self.content = content

        # S はこのメソッドでのみバインドされた型変数です
        def first(self, x: list[S]) -> S:
            return x[0]

        def pair_with_first(self, x: list[S]) -> tuple[S, T]:
            return (x[0], self.content)

    box = PairedBox("asdf")
    reveal_type(box.first([1, 2, 3]))  # 明らかにされた型は "builtins.int" です
    reveal_type(box.pair_with_first([1, 2, 3]))  # 明らかにされた型は "tuple[builtins.int, builtins.str]" です

特に、``self`` 引数もジェネリックにすることができ、メソッドがアクセス時点で既知の最も正確な型を返すことができます。 この方法で、セッター メソッドのチェーンを型チェックできます。

.. code-block:: python

   from typing import TypeVar

   T = TypeVar('T', bound='Shape')

   class Shape:
       def set_scale(self: T, scale: float) -> T:
           self.scale = scale
           return self

   class Circle(Shape):
       def set_radius(self, r: float) -> 'Circle':
           self.radius = r
           return self

   class Square(Shape):
       def set_width(self, w: float) -> 'Square':
           self.width = w
           return self

   circle: Circle = Circle().set_scale(0.5).set_radius(2.7)
   square: Square = Square().set_scale(0.5).set_width(3.2)

ジェネリック ``self`` を使用しない場合、``set_scale`` の戻り値の型は ``Shape`` であり、``set_radius`` や ``set_width`` は定義されていないため、最後の 2 行は正しく型チェックできません。

他の使用例としては、コピーやデシリアライズのファクトリ メソッドがあります。 クラス メソッドの場合、:py:class:`type` を使用してジェネリック ``cls`` を定義することもできます。

.. code-block:: python

   from typing import Optional, TypeVar, Type

   T = TypeVar('T', bound='Friend')

   class Friend:
       other: Optional["Friend"] = None

       @classmethod
       def make_pair(cls: type[T]) -> tuple[T, T]:
           a, b = cls(), cls()
           a.other = b
           b.other = a
           return a, b

   class SuperFriend(Friend):
       pass

   a, b = SuperFriend.make_pair()

ジェネリック ``self`` を使用してメソッドをオーバーライドする場合、ジェネリック ``self`` も返すか、現在のクラスのインスタンスを返す必要があります。 後者の場合、このメソッドをすべての将来のサブクラスで実装する必要があります。

コピーやデシリアライズ メソッドの実装が self の実際の型を返すことを型チェッカーが常に検証するわけではないことにも注意してください。 したがって、これらのメソッド内 (呼び出し元ではなく) で型チェッカーを黙らせる必要がある場合があります。これには、``Any`` 型や ``# type: ignore`` コメントを使用することが考えられます。

typing.Self を使用した自動 self 型
******************************************************************************************

上記のパターンは非常に一般的であるため、:pep:`673` でより簡単な構文が導入されました。

明示的な注釈の代わりに、特別な型 ``typing.Self`` を使用できます。 これは、現在のクラスを上限とする型変数に自動的に変換され、``self`` (またはクラス メソッドの ``cls``) に注釈を付ける必要はありません。

前のセクションの例を ``typing.Self`` を使用して書き直すと次のようになります。

.. code-block:: python

   from typing import Self

   class Friend:
       other: Self | None = None

       @classmethod
       def make_pair(cls) -> tuple[Self, Self]:
           a, b = cls(), cls()
           a.other = b
           b.other = a
           return a, b

   class SuperFriend(Friend):
       pass

   a, b = SuperFriend.make_pair()

これは、明示的な型変数を使用するよりもコンパクトです。 また、メソッドに加えて属性注釈にも ``Self`` を使用できます。

.. note::

   この機能を Python 3.11 より前のバージョンで使用するには、``Self`` を ``typing_extensions`` (バージョン 4.0 以降) からインポートする必要があります。

.. _variance-of-generics:

ジェネリック型の分散
******************************************************************************************

サブタイプ間のサブタイプ関係に関して、ジェネリック型には 3 つの主要な種類があります。 これらは、不変、共変、および反変です。 ``Animal`` と ``Bear`` の 2 つの型があり、``Bear`` が ``Animal`` のサブタイプであると仮定すると、これらは次のように定義されます。

* ジェネリック クラス ``MyCovGen[T]`` は、型パラメーター ``T`` に関して共変であると呼ばれます。 ``MyCovGen[Bear]`` は ``MyCovGen[Animal]`` のサブタイプです。 これは、最も直感的な形式の分散です。
* ジェネリック クラス ``MyContraGen[T]`` は、型パラメーター ``T`` に関して反変であると呼ばれます。 ``MyContraGen[Animal]`` は ``MyContraGen[Bear]`` のサブタイプです。
* ジェネリック クラス ``MyInvGen[T]`` は、上記のいずれでもない場合、``T`` に関して不変であると呼ばれます。

いくつかの簡単な例でこれを説明しましょう。

.. code-block:: python

    # 以下のクラスは、次の例で使用します
    class Shape: ...
    class Triangle(Shape): ...
    class Square(Shape): ...

* :py:class:`~typing.Sequence` や :py:class:`~typing.FrozenSet` などのほとんどの不変コンテナーは共変です。 :py:data:`~typing.Union` もすべての変数で共変です。 ``Union[Triangle, int]`` は ``Union[Shape, int]`` のサブタイプです。

  .. code-block:: python

    def count_lines(shapes: Sequence[Shape]) -> int:
        return sum(shape.num_sides for shape in shapes)

    triangles: Sequence[Triangle]
    count_lines(triangles)  # OK

    def foo(triangle: Triangle, num: int):
        shape_or_number: Union[Shape, int]
        # Triangle は Shape であり、Shape は有効な Union[Shape, int] です
        shape_or_number = triangle

  共変性は比較的直感的に感じられますが、反変性と不変性は理解するのが難しい場合があります。

* :py:data:`~typing.Callable` は、引数の型に関して反変的に動作する型の例です。 つまり、``Callable[[Shape], int]`` は ``Callable[[Triangle], int]`` のサブタイプです。 これは、``Shape`` が ``Triangle`` のスーパータイプであるにもかかわらずです。 これを理解するには、次のように考えます。

  .. code-block:: python

    def cost_of_paint_required(
        triangle: Triangle,
        area_calculator: Callable[[Triangle], float]
    ) -> float:
        return area_calculator(triangle) * DOLLAR_PER_SQ_FT

    # これは簡単に動作します
    def area_of_triangle(triangle: Triangle) -> float: ...
    cost_of_paint_required(triangle, area_of_triangle)  # OK

    # しかし、これも動作します!
    def area_of_any_shape(shape: Shape) -> float: ...
    cost_of_paint_required(triangle, area_of_any_shape)  # OK

  ``cost_of_paint_required`` は三角形の面積を計算できるコールバックを必要とします。 任意の形状の面積を計算できるコールバックを提供する場合 (三角形に限定されない)、すべてが正常に動作します。

* :py:class:`~typing.List` は不変のジェネリック型です。 一見、:py:class:`~typing.Sequence` のように共変であると思われるかもしれませんが、次のコードを考えてみてください。

  .. code-block:: python

     class Circle(Shape):
         # rotate メソッドは Circle にのみ定義されており、Shape には定義されていません
         def rotate(self): ...

     def add_one(things: list[Shape]) -> None:
         things.append(Shape())

     my_circles: list[Circle] = []
     add_one(my_circles)     # これは安全に見えるかもしれませんが...
     my_circles[-1].rotate()  # ...これは失敗します。my_circles[0] は Circle ではなく Shape です

  不変型の別の例は :py:class:`~typing.Dict` です。 ほとんどの可変コンテナーは不変です。

デフォルトでは、すべてのユーザー定義ジェネリックは不変です。 特定のジェネリック クラスを共変または反変として宣言するには、``covariant`` または ``contravariant`` という特別なキーワード引数で定義された型変数を使用します。 例えば:

.. code-block:: python

   from typing import Generic, TypeVar

   T_co = TypeVar('T_co', covariant=True)

   class Box(Generic[T_co]):  # この型は共変であると宣言されています
       def __init__(self, content: T_co) -> None:
           self._content = content

       def get_content(self) -> T_co:
           return self._content

   def look_into(box: Box[Animal]): ...

   my_box = Box(Cat())
   look_into(my_box)  # OK、ただし Box が T で不変である場合はエラーになります

.. _type-variable-upper-bound:

上限を持つ型変数
******************************************************************************************

デフォルトでは、型変数は任意の型に置き換えることができます。 これは、``T`` 型のオブジェクトに対して安全にできることがほとんどないことを意味します。 ``T`` について何も知らないからです。

したがって、型変数が特定の型のサブタイプである値に制限できることがよくあります。

このような型は型変数の上限と呼ばれ、:py:class:`~typing.TypeVar` の ``bound=...`` キーワード引数で指定されます。

.. code-block:: python

    from typing import TypeVar, SupportsAbs

    T = TypeVar('T', bound=SupportsAbs[float])

このような型変数 ``T`` を使用するジェネリック関数の定義では、``T`` によって表される型はその上限のサブタイプであると見なされるため、関数は ``T`` 型の値に対して上限のメソッドを使用できます。

.. code-block:: python

    def largest_in_absolute_value(*xs: T) -> T:
        return max(xs, key=abs)  # OK、T は SupportsAbs[float] のサブタイプです。

このような関数の呼び出しでは、型 ``T`` はその上限のサブタイプである型に置き換える必要があります。 上記の例を続けます。

.. code-block:: python

    largest_in_absolute_value(-3.5, 2)   # OK、型は float です
    largest_in_absolute_value(5+6j, 7)   # OK、型は complex です
    largest_in_absolute_value('a', 'b')  # エラー: "largest_in_absolute_value" の型変数 "T" の値は "str" にはなれません

ジェネリック クラスの型パラメーターにも上限があり、同様に型パラメーターの有効な値を制限します。

.. _type-variable-value-restriction:

制約付きの型変数
******************************************************************************************

場合によっては、型変数が特定の型のセットにのみ値を取るように制限することが役立つことがあります。 この機能は少し複雑であり、上記のように上限を使用して機能させることができる場合は避けるべきです。

例として、値が ``str`` と ``bytes`` のみである型変数があります。

.. code-block:: python

   from typing import TypeVar

   AnyStr = TypeVar('AnyStr', str, bytes)

実際には、これは :py:data:`~typing.AnyStr` が :py:mod:`typing` で定義されているほど一般的な型変数です。

:py:data:`~typing.AnyStr` を使用して、2 つの文字列またはバイト オブジェクトを連結できる関数を定義できますが、他の引数の型では呼び出せません。

.. code-block:: python

   from typing import AnyStr

   def concat(x: AnyStr, y: AnyStr) -> AnyStr:
       return x + y

   concat('a', 'b')    # OK
   concat(b'a', b'b')  # OK
   concat(1, 2)        # エラー!

重要なのは、これは共用体型とは異なり、``str`` と ``bytes`` の組み合わせは受け入れられないことです。

.. code-block:: python

   concat('string', b'bytes')   # エラー!

この場合、文字列とバイト オブジェクトを連結することはできないため、これはまさに私たちが望むものです。 ``Union`` を使用しようとすると、型チェッカーはこの可能性について警告します。

.. code-block:: python

   def union_concat(x: Union[str, bytes], y: Union[str, bytes]) -> Union[str, bytes]:
       return x + y  # エラー: str と bytes を連結できません

``concat()`` を ``str`` のサブタイプで呼び出す場合のもう 1 つの興味深い特別なケースです。

.. code-block:: python

    class S(str): pass

    ss = concat(S('foo'), S('bar'))
    reveal_type(ss)  # 明らかにされた型は "builtins.str" です

``ss`` の型が ``S`` であると予想するかもしれませんが、実際の型は ``str`` です。 サブタイプは型変数の有効な値の 1 つに昇格されます。この場合は ``str`` です。

したがって、これは Java などの言語の *制約付き量化* とは微妙に異なります。 これらの言語では、戻り値の型は ``S`` になります。 型チェッカーがこれを実装する方法は、上記の例で ``concat`` が実際に ``str`` のインスタンスを返すため、実際には ``concat`` に対してまさに私たちが望むことを行います。

.. code-block:: python

    >>> print(type(ss))
    <class 'str'>

ジェネリック クラスを定義する場合にも、制約付きの :py:class:`~typing.TypeVar` を使用できます。 例えば、正規表現は文字列またはバイト パターンに基づくことができるため、:py:func:`re.compile` の戻り値に :py:class:`Pattern[AnyStr] <typing.Pattern>` を使用できます。

型変数には、値の制限 (参照 :ref:`type-variable-upper-bound`) と上限の両方を持つことはできません。

.. _declaring-decorators:

デコレータの宣言
******************************************************************************************

デコレータは通常、関数を引数として受け取り、別の関数を返す関数です。 型の観点からこの動作を説明するのは少し難しい場合があります。 ``TypeVar`` と *パラメーター仕様* と呼ばれる特別な種類の型変数を使用する方法を示します。

次のデコレータがあると仮定します。これはまだ型注釈されておらず、元の関数のシグネチャを保持し、装飾された関数の名前を出力するだけです。

.. code-block:: python

   def printing_decorator(func):
       def wrapper(*args, **kwds):
           print("Calling", func)
           return func(*args, **kwds)
       return wrapper

そして、関数 ``add_forty_two`` を装飾するために使用します。

.. code-block:: python

   # 装飾された関数。
   @printing_decorator
   def add_forty_two(value: int) -> int:
       return value + 42

   a = add_forty_two(3)

``printing_decorator`` が型注釈されていないため、次のことは型チェックされません。

.. code-block:: python

   reveal_type(a)        # 明らかにされた型は "Any" です
   add_forty_two('foo')  # 型チェッカー エラーはありません :(

これは残念な状態です!

デコレータに注釈を付ける方法を次に示します。

.. code-block:: python

   from typing import Any, Callable, TypeVar, cast

   F = TypeVar('F', bound=Callable[..., Any])

   # シグネチャを保持するデコレータ。
   def printing_decorator(func: F) -> F:
       def wrapper(*args, **kwds):
           print("Calling", func)
           return func(*args, **kwds)
       return cast(F, wrapper)

   @printing_decorator
   def add_forty_two(value: int) -> int:
       return value + 42

   a = add_forty_two(3)
   reveal_type(a)      # 明らかにされた型は "builtins.int" です
   add_forty_two('x')  # "add_forty_two" の引数 1 の型が一致しません。予期される型は "int" です

これにはまだいくつかの欠点があります。 まず、型チェッカーに ``wrapper()`` が ``func`` と同じシグネチャを持っていることを納得させるために、安全でない :py:func:`~typing.cast` を使用する必要があります。

第二に、``wrapper()`` 関数は厳密に型チェックされませんが、ラッパー関数は通常十分に小さいため、これは大きな問題ではありません。 これは、``printing_decorator()`` の ``return`` ステートメントで :py:func:`~typing.cast` 呼び出しがある理由でもあります。

ただし、パラメーター仕様 (:py:class:`~typing.ParamSpec`) を使用して、より正確な型注釈を行うことができます。

.. code-block:: python

   from typing import Callable, TypeVar
   from typing_extensions import ParamSpec

   P = ParamSpec('P')
   T = TypeVar('T')

   def printing_decorator(func: Callable[P, T]) -> Callable[P, T]:
       def wrapper(*args: P.args, **kwds: P.kwargs) -> T:
           print("Calling", func)
           return func(*args, **kwds)
       return wrapper

パラメーター仕様を使用すると、入力関数のシグネチャを変更するデコレータを記述することもできます。

.. code-block:: python

   from typing import Callable, TypeVar
   from typing_extensions import ParamSpec

   P = ParamSpec('P')
   T = TypeVar('T')

   # 'P' を戻り値の型で再利用しますが、'T' を 'str' に置き換えます
   def stringify(func: Callable[P, T]) -> Callable[P, str]:
       def wrapper(*args: P.args, **kwds: P.kwargs) -> str:
           return str(func(*args, **kwds))
       return wrapper

   @stringify
   def add_forty_two(value: int) -> int:
       return value + 42

   a = add_forty_two(3)
   reveal_type(a)      # 明らかにされた型は "builtins.str" です
   add_forty_two('x')  # エラー: "add_forty_two" の引数 1 の型が一致しません。予期される型は "int" です

または引数を挿入します。

.. code-block:: python

    from typing import Callable, TypeVar
    from typing_extensions import Concatenate, ParamSpec

    P = ParamSpec('P')
    T = TypeVar('T')

    def printing_decorator(func: Callable[P, T]) -> Callable[Concatenate[str, P], T]:
        def wrapper(msg: str, /, *args: P.args, **kwds: P.kwargs) -> T:
            print("Calling", func, "with", msg)
            return func(*args, **kwds)
        return wrapper

    @printing_decorator
    def add_forty_two(value: int) -> int:
        return value + 42

    a = add_forty_two('three', 3)

.. _decorator-factories:

デコレータ ファクトリ
------------------------------------------------------------------------------------------

引数を取り、デコレータを返す関数 (セカンドオーダー デコレータとも呼ばれます) も、ジェネリックを介して同様にサポートされます。

.. code-block:: python

    from typing import Any, Callable, TypeVar

    F = TypeVar('F', bound=Callable[..., Any])

    def route(url: str) -> Callable[[F], F]:
        ...

    @route(url='/')
    def index(request: Any) -> str:
        return 'Hello world'

同じデコレータが引数なしの呼び出しと引数付きの呼び出しの両方をサポートする場合があります。 これは、:py:func:`@overload <typing.overload>` と組み合わせることで実現できます。

.. code-block:: python

    from typing import Any, Callable, Optional, TypeVar, overload

    F = TypeVar('F', bound=Callable[..., Any])

    # 裸のデコレータの使用
    @overload
    def atomic(__func: F) -> F: ...
    # 引数付きのデコレータ
    @overload
    def atomic(*, savepoint: bool = True) -> Callable[[F], F]: ...

    # 実装
    def atomic(__func: Optional[Callable[..., Any]] = None, *, savepoint: bool = True):
        def decorator(func: Callable[..., Any]):
            ...  # コードはここに記述します
        if __func is not None:
            return decorator(__func)
        else:
            return decorator

    # 使用法
    @atomic
    def func1() -> None: ...

    @atomic(savepoint=False)
    def func2() -> None: ...

ジェネリック プロトコル
******************************************************************************************

プロトコルもジェネリックにすることができます (参照 :ref:`protocol-types`)。 いくつかの :ref:`predefined protocols <predefined_protocols>` はジェネリックであり、:py:class:`Iterable[T] <typing.Iterable>` など、追加のジェネリック プロトコルを定義できます。 ジェネリック プロトコルは、通常のジェネリック クラスのルールにほぼ従います。 例:

.. code-block:: python

   from typing import TypeVar
   from typing_extensions import Protocol

   T = TypeVar('T')

   class Box(Protocol[T]):
       content: T

   def do_stuff(one: Box[str], other: Box[bytes]) -> None:
       ...

   class StringWrapper:
       def __init__(self, content: str) -> None:
           self.content = content

   class BytesWrapper:
       def __init__(self, content: bytes) -> None:
           self.content = content

   do_stuff(StringWrapper('one'), BytesWrapper(b'other'))  # OK

   x: Box[float] = ...
   y: Box[int] = ...
   x = y  # エラー -- Box は不変です

``class ClassName(Protocol[T])`` は ``class ClassName(Protocol, Generic[T])`` の省略形として許可されていることに注意してください。 :pep:`PEP 544: Generic protocols <544#generic-protocols>` に従います。

ジェネリック プロトコルと通常のジェネリック クラスの主な違いは、プロトコルのジェネリック型変数の宣言された分散が、プロトコル定義での使用方法に対してチェックされることです。 この例のプロトコルは拒否されます。型変数 ``T`` は戻り値の型として共変的に使用されますが、型変数は不変です。

.. code-block:: python

   from typing import Protocol, TypeVar

   T = TypeVar('T')

   class ReadOnlyBox(Protocol[T]):  # エラー: 共変の型変数が期待されるプロトコルで不変の型変数 "T" が使用されました
       def content(self) -> T: ...

この例では、共変の型変数を正しく使用しています。

.. code-block:: python

   from typing import Protocol, TypeVar

   T_co = TypeVar('T_co', covariant=True)

   class ReadOnlyBox(Protocol[T_co]):  # OK
       def content(self) -> T_co: ...

   ax: ReadOnlyBox[float] = ...
   ay: ReadOnlyBox[int] = ...
   ax = ay  # OK -- ReadOnlyBox は共変です

分散の詳細については、:ref:`variance-of-generics` を参照してください。

ジェネリック プロトコルも再帰的にすることができます。 例:

.. code-block:: python

   T = TypeVar('T')

   class Linked(Protocol[T]):
       val: T
       def next(self) -> 'Linked[T]': ...

   class L:
       val: int
       def next(self) -> 'L': ...

   def last(seq: Linked[T]) -> T: ...

   result = last(L())
   reveal_type(result)  # 明らかにされた型は "builtins.int" です

.. _generic-type-aliases:

ジェネリック型エイリアス
******************************************************************************************

型エイリアスはジェネリックにすることができます。 この場合、2 つの方法で使用できます。 添字付きエイリアスは、置換された型変数を持つ元の型と同等であるため、型引数の数はジェネリック型エイリアスの自由型変数の数と一致する必要があります。 添字なしエイリアスは、自由変数が ``Any`` に置き換えられた元の型として扱われます。 例 (:pep:`PEP 484: Type aliases <484#type-aliases>` に従います):

.. code-block:: python

    from typing import TypeVar, Iterable, Union, Callable

    S = TypeVar('S')

    TInt = tuple[int, S]
    UInt = Union[S, int]
    CBack = Callable[..., S]

    def response(query: str) -> UInt[str]:  # Union[str, int] と同じ
        ...
    def activate(cb: CBack[S]) -> S:        # Callable[..., S] と同じ
        ...
    table_entry: TInt  # tuple[int, Any] と同じ

    T = TypeVar('T', int, float, complex)

    Vec = Iterable[tuple[T, T]]

    def inproduct(v: Vec[T]) -> T:
        return sum(x*y for x, y in v)

    def dilate(v: Vec[T], scale: T) -> Vec[T]:
        return ((x * scale, y * scale) for x, y in v)

    v1: Vec[int] = []      # Iterable[tuple[int, int]] と同じ
    v2: Vec = []           # Iterable[tuple[Any, Any]] と同じ
    v3: Vec[int, int] = [] # エラー: 無効なエイリアス、型引数が多すぎます!

型エイリアスは、他の名前と同様にモジュールからインポートできます。 エイリアスは別のエイリアスをターゲットにすることもできますが、エイリアスの複雑なチェーンを構築することはお勧めしません。 これはコードの可読性を損ない、エイリアスを使用する目的を損ないます。 例:

.. code-block:: python

    from typing import TypeVar, Generic, Optional
    from example1 import AliasType
    from example2 import Vec

    # AliasType と Vec は型エイリアスです (上記の Vec として定義されています)

    def fun() -> AliasType:
        ...

    T = TypeVar('T')

    class NewVec(Vec[T]):
        ...

    for i, j in NewVec[int]():
        ...

    OIntVec = Optional[Vec[int]]

ジェネリック エイリアスでの型変数の上限または値の使用は、ジェネリック クラス/関数での使用と同じ効果があります。

ジェネリック クラスの内部
******************************************************************************************

ジェネリック クラスをインデックスすると、実行時に何が起こるか疑問に思うかもしれません。 インデックス作成は、インスタンス化時に元のクラスのインスタンスを返す元のクラスの *ジェネリック エイリアス* を返します。

.. code-block:: python

   >>> from typing import TypeVar, Generic
   >>> T = TypeVar('T')
   >>> class Stack(Generic[T]): ...
   >>> Stack
   __main__.Stack
   >>> Stack[int]
   __main__.Stack[int]
   >>> instance = Stack[int]()
   >>> instance.__class__
   __main__.Stack

ジェネリック エイリアスは、実際のクラスと同様にインスタンス化またはサブクラス化できますが、上記の例は、型変数が実行時に消去されることを示しています。 ジェネリック ``Stack`` インスタンスは単なる通常の Python オブジェクトであり、インデックス作成操作をオーバーロードする以外に、ジェネリックであるための追加の実行時オーバーヘッドやマジックはありません。

Python 3.8 以前では、組み込み型 :py:class:`list`、:py:class:`dict` などはインデックス作成をサポートしていないことに注意してください。 これが、:py:mod:`typing` モジュールに :py:class:`~typing.List`、:py:class:`~typing.Dict` などのエイリアスがある理由です。 これらのエイリアスにインデックスを付けると、より最近のバージョンの Python でターゲット クラスを直接インデックス付けすることによって構築されたジェネリック エイリアスに似たジェネリック エイリアスが得られます。

.. code-block:: python

   >>> # Python 3.8 以前にのみ関連します
   >>> # Python 3.9 以降の場合、`list[int]` 構文を使用することをお勧めします
   >>> from typing import List
   >>> List[int]
   typing.List[int]

``typing`` のジェネリック エイリアスはインスタンスの構築をサポートしていないことに注意してください。

.. code-block:: python

   >>> from typing import List
   >>> List[int]()
   Traceback (most recent call last):
   ...
   TypeError: Type List cannot be instantiated; use list() instead

クレジット
******************************************************************************************

このドキュメントは、`mypy documentation <https://mypy.readthedocs.io/en/stable/>`_ に基づいています。
