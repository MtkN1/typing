.. _`generics`:

ジェネリクス
==========================================================================================

はじめに
------------------------------------------------------------------------------------------

コンテナに保持されているオブジェクトに関する型情報は、ジェネリックな方法では静的に推測することができないため、抽象基底クラスは、コンテナ要素の予想される型を示すためのサブスクリプションをサポートするように拡張されました。 例::

  from collections.abc import Mapping

  def notify_by_email(employees: set[Employee], overrides: Mapping[str, str]) -> None: ...

ジェネリクスは、``TypeVar`` と呼ばれる ``typing`` で利用可能なファクトリを使用してパラメータ化することができます。 例::

  from collections.abc import Sequence
  from typing import TypeVar

  T = TypeVar('T')      # 型変数を宣言する

  def first(l: Sequence[T]) -> T:   # ジェネリック関数
      return l[0]

または、Python 3.12 (:pep:`695`) 以降では、ジェネリック関数の新しい構文を使用します::

  from collections.abc import Sequence

  def first[T](l: Sequence[T]) -> T:   # ジェネリック関数
      return l[0]

この 2 つの構文は同等です。
いずれの場合も、返される値はコレクションによって保持される要素と一貫しているという契約です。

``TypeVar()`` 式は常に変数に直接代入されなければなりません (それはより大きな式の一部として使用されるべきではありません)。 ``TypeVar()`` の引数は、代入される変数名と等しい文字列でなければなりません。 型変数は再定義してはなりません。

``TypeVar`` は、パラメトリック型を可能な型の固定セットに制約することをサポートします (注: これらの型は型変数によってパラメータ化することはできません)。 例えば、``str`` と ``bytes`` のみを範囲とする型変数を定義することができます。 デフォルトでは、型変数はすべての可能な型を範囲とします。 型変数を制約する例::

  from typing import TypeVar

  AnyStr = TypeVar('AnyStr', str, bytes)

  def concat(x: AnyStr, y: AnyStr) -> AnyStr:
      return x + y

または組み込みの構文を使用します (3.12 以降)::

  def concat[AnyStr: (str, bytes)](x: AnyStr, y: AnyStr) -> AnyStr:
      return x + y

関数 ``concat`` は、2 つの ``str`` 引数または 2 つの ``bytes`` 引数のいずれかで呼び出すことができますが、``str`` と ``bytes`` の混在した引数では呼び出すことはできません。

制約がある場合は、少なくとも 2 つの制約があるべきです。 単一の制約を指定することは許可されていません。

型変数によって制約された型のサブタイプは、型変数のコンテキストでそれぞれの明示的にリストされた基本型として扱われるべきです。 次の例を考えてみましょう::

  class MyStr(str): ...

  x = concat(MyStr('apple'), MyStr('pie'))

この呼び出しは有効ですが、型変数 ``AnyStr`` は ``str`` に設定され、``MyStr`` には設定されません。 実際には、``x`` に代入される戻り値の推論された型も ``str`` になります。

さらに、``Any`` はすべての型変数に対して有効な値です。 次の例を考えてみましょう::

  def count_truthy(elements: list[Any]) -> int:
      return sum(1 for elem in elements if elem)

これはジェネリック表記を省略して単に ``elements: list`` と言うのと同じです。


ユーザー定義のジェネリック型
------------------------------------------------------------------------------------------

``Generic`` 基底クラスを含めることで、ユーザー定義のクラスをジェネリックとして定義することができます。 例::

  from typing import TypeVar, Generic
  from logging import Logger

  T = TypeVar('T')

  class LoggedVar(Generic[T]):
      def __init__(self, value: T, name: str, logger: Logger) -> None:
          self.name = name
          self.logger = logger
          self.value = value

      def set(self, new: T) -> None:
          self.log('Set ' + repr(self.value))
          self.value = new

      def get(self) -> T:
          self.log('Get ' + repr(self.value))
          return self.value

      def log(self, message: str) -> None:
          self.logger.info('{}: {}'.format(self.name, message))

または、Python 3.12 以降では、ジェネリッククラスの新しい構文を使用します::

  class LoggedVar[T]:
      # 前の例と同じメソッド

これにより、暗黙的に ``Generic[T]`` が基底クラスとして追加され、型チェッカーは 2 つをほぼ同等に扱うべきです (分散を除く、以下参照)。

基底クラスとしての ``Generic[T]`` は、クラス ``LoggedVar`` が単一の型パラメータ ``T`` を取ることを定義します。 これにより、クラス本体内で ``T`` を型として使用することもできます。

``Generic`` 基底クラスは ``__getitem__`` を定義するメタクラスを使用しているため、``LoggedVar[t]`` は型として有効です::

  from collections.abc import Iterable

  def zero_all_vars(vars: Iterable[LoggedVar[int]]) -> None:
      for var in vars:
          var.set(0)

ジェネリック型は任意の数の型変数を持つことができ、型変数は制約される場合があります。 これは有効です::

  from typing import TypeVar, Generic
  ...

  T = TypeVar('T')
  S = TypeVar('S')

  class Pair(Generic[T, S]):
      ...

``Generic`` の各型変数引数は異なるものでなければなりません。 したがって、これは無効です::

  from typing import TypeVar, Generic
  ...

  T = TypeVar('T')

  class Pair(Generic[T, T]):   # 無効
      ...

``Generic[T]`` 基底クラスは、他のジェネリッククラスをサブクラス化し、そのパラメータに型変数を指定する場合、単純なケースでは冗長です::

  from typing import TypeVar
  from collections.abc import Iterator

  T = TypeVar('T')

  class MyIter(Iterator[T]):
      ...

そのクラス定義は次のものと同等です::

  class MyIter(Iterator[T], Generic[T]):
      ...

``Generic`` を使用した多重継承が可能です::

  from typing import TypeVar, Generic
  from collections.abc import Sized, Iterable, Container

  T = TypeVar('T')

  class LinkedList(Sized, Generic[T]):
      ...

  K = TypeVar('K')
  V = TypeVar('V')

  class MyMapping(Iterable[tuple[K, V]],
                  Container[tuple[K, V]],
                  Generic[K, V]):
      ...

型パラメータにデフォルト値がない限り、型パラメータを指定せずにジェネリッククラスをサブクラス化すると、各位置に ``Any`` が仮定されます。 次の例では、``MyIterable`` はジェネリックではありませんが、暗黙的に ``Iterable[Any]`` から継承されます::

  from collections.abc import Iterable

  class MyIterable(Iterable):  # Iterable[Any] と同じ
      ...

ジェネリックメタクラスはサポートされていません。

.. _`typevar-scoping`:

型変数のスコープルール
------------------------------------------------------------------------------------------

型変数は通常の名前解決ルールに従います。
ただし、静的型チェックコンテキストにはいくつかの特別なケースがあります:

* ジェネリック関数で使用される型変数は、同じコードブロック内で異なる型を表すと推測されることがあります。 例::

    from typing import TypeVar, Generic

    T = TypeVar('T')

    def fun_1(x: T) -> T: ...  # ここでの T
    def fun_2(x: T) -> T: ...  # そしてここでの T は異なる可能性があります

    fun_1(1)                   # これは OK です。T は int と推測されます
    fun_2('a')                 # これも OK です。今度は T は str です

* ジェネリッククラスのメソッドで使用される型変数が、このクラスをパラメータ化する変数の 1 つと一致する場合、その変数に常にバインドされます。 例::

    from typing import TypeVar, Generic

    T = TypeVar('T')

    class MyClass(Generic[T]):
        def meth_1(self, x: T) -> T: ...  # ここでの T
        def meth_2(self, x: T) -> T: ...  # そしてここでの T は常に同じです

    a: MyClass[int] = MyClass()
    a.meth_1(1)    # OK
    a.meth_2('a')  # これはエラーです!

* クラスをパラメータ化する変数と一致しないメソッドで使用される型変数は、その変数でジェネリック関数になります::

    T = TypeVar('T')
    S = TypeVar('S')
    class Foo(Generic[T]):
        def method(self, x: T, y: S) -> S:
            ...

    x: Foo[int] = Foo()
    y = x.method(0, "abc")  # y の推論された型は str です

* ジェネリック関数の本体やメソッド定義以外のクラス本体に未バインドの型変数が現れるべきではありません::

    T = TypeVar('T')
    S = TypeVar('S')

    def a_fun(x: T) -> None:
        # これは OK です
        y: list[T] = []
        # しかし、以下はエラーです!
        y: list[S] = []

    class Bar(Generic[T]):
        # これもエラーです
        an_attr: list[S] = []

        def do_something(self, x: S) -> S:  # これは OK です
            ...

* ジェネリック関数内に現れるジェネリッククラス定義は、そのジェネリック関数をパラメータ化する型変数を使用してはなりません::

    def a_fun(x: T) -> None:

        # これは OK です
        a_list: list[T] = []
        ...

        # これは違法です
        class MyGeneric(Generic[T]):
            ...

* 外部クラスの型変数のスコープは内部クラスをカバーしないため、別のジェネリッククラスにネストされたジェネリッククラスは同じ型変数を使用できません::

    T = TypeVar('T')
    S = TypeVar('S')

    class Outer(Generic[T]):
        class Bad(Iterable[T]):       # エラー
            ...
        class AlsoBad:
            x: list[T]  # これもエラー

        class Inner(Iterable[S]):     # OK
            ...
        attr: Inner[T]  # これも OK


ジェネリッククラスのインスタンス化と型消去
------------------------------------------------------------------------------------------

ユーザー定義のジェネリッククラスはインスタンス化できます。 ``Generic[T]`` を継承する ``Node`` クラスを記述するとします::

  from typing import TypeVar, Generic

  T = TypeVar('T')

  class Node(Generic[T]):
      ...

``Node`` インスタンスを作成するには、通常のクラスと同様に ``Node()`` を呼び出します。 実行時には、インスタンスの型 (クラス) は ``Node`` になります。
しかし、型チェッカーにとってはどのような型を持つのでしょうか? 答えは、呼び出し時に利用可能な情報の量によって異なります。 コンストラクタ (``__init__`` または ``__new__``) がそのシグネチャで ``T`` を使用し、対応する引数値が渡された場合、対応する引数の型が置き換えられます。 それ以外の場合、型パラメータのデフォルト値 (またはデフォルトが提供されていない場合は ``Any``) が仮定されます。 例::

  from typing import TypeVar, Generic

  T = TypeVar('T')

  class Node(Generic[T]):
      x: T # インスタンス属性 (以下参照)
      def __init__(self, label: T | None = None) -> None:
          ...

  x = Node('')  # 推論された型は Node[str]
  y = Node(0)   # 推論された型は Node[int]
  z = Node()    # 推論された型は Node[Any]

推論された型が ``[Any]`` を使用する場合でも、意図された型がより具体的である場合は、以下のようにアノテーションを使用して変数の型を強制することができます::

  # (前の例から続く)
  a: Node[int] = Node()
  b: Node[str] = Node()

または、特定の具体的な型をインスタンス化することもできます::

  # (前の例から続く)
  p = Node[int]()
  q = Node[str]()
  r = Node[int]('')  # エラー
  s = Node[str](0)   # エラー

``p`` と ``q`` の実行時の型 (クラス) は依然として ``Node`` であることに注意してください。 ``Node[int]`` と ``Node[str]`` は区別可能なクラスオブジェクトですが、それらをインスタンス化して作成されたオブジェクトの実行時のクラスはその区別を記録しません。 この動作は「型消去」と呼ばれ、ジェネリクスを持つ言語 (例: Java、TypeScript) では一般的なプラクティスです。

ジェネリッククラス (パラメータ化されているかどうかにかかわらず) を使用して属性にアクセスすると、型チェックエラーが発生します。 クラス定義本体の外部では、クラス属性を代入することはできず、同じ名前のインスタンス属性を持たないクラスインスタンスを通じてアクセスすることでのみクラス属性を参照できます::

  # (前の例から続く)
  Node[int].x = 1  # エラー
  Node[int].x      # エラー
  Node.x = 1       # エラー
  Node.x           # エラー
  type(p).x        # エラー
  p.x              # OK (int と評価される)
  Node[int]().x    # OK (int と評価される)
  p.x = 1          # OK、ただしインスタンス属性に代入される

``Mapping`` や ``Sequence`` などの抽象コレクションのジェネリックバージョンや、``List``、``Dict``、``Set``、``FrozenSet`` などの組み込みクラスのジェネリックバージョンはインスタンス化できません。 ただし、それらの具体的なユーザー定義サブクラスや具体的なコレクションのジェネリックバージョンはインスタンス化できます::

  data = DefaultDict[int, bytes]()

静的型と実行時クラスを混同しないように注意してください。
この場合でも型は消去され、上記の式は単なる省略形です::

  data: DefaultDict[int, bytes] = collections.defaultdict()

添字付きクラス (例: ``Node[int]``) を直接式で使用することは推奨されません。 代わりに型エイリアス (例: ``IntNode = Node[int]``) を使用することが推奨されます。 (まず、添字付きクラス (例: ``Node[int]``) を作成することには実行時のコストがあります。 第二に、型エイリアスを使用する方が読みやすいです。)


基底クラスとしての任意のジェネリック型
------------------------------------------------------------------------------------------

``Generic[T]`` は基底クラスとしてのみ有効です。 それは適切な型ではありません。
ただし、上記の例の ``LinkedList[T]`` などのユーザー定義のジェネリック型や、``list[T]`` や ``Iterable[T]`` などの組み込みのジェネリック型や ABC は、型としても基底クラスとしても有効です。 例えば、型引数を特化する ``dict`` のサブクラスを定義できます::

  class Node:
      ...

  class SymbolTable(dict[str, list[Node]]):
      def push(self, name: str, node: Node) -> None:
          self.setdefault(name, []).append(node)

      def pop(self, name: str) -> Node:
          return self[name].pop()

      def lookup(self, name: str) -> Node | None:
          nodes = self.get(name)
          if nodes:
              return nodes[-1]
          return None

``SymbolTable`` は ``dict`` のサブクラスであり、``dict[str, list[Node]]`` のサブタイプです。

ジェネリック基底クラスが型引数として型変数を持つ場合、定義されたクラスはジェネリックになります。 例えば、反復可能でコンテナであるジェネリック ``LinkedList`` クラスを定義できます::

  from typing import TypeVar
  from collections.abc import Iterable, Container

  T = TypeVar('T')

  class LinkedList(Iterable[T], Container[T]):
      ...

これで ``LinkedList[int]`` は有効な型です。 ``T`` を ``Generic[...]`` 内で複数回使用しない限り、基底クラスリスト内で ``T`` を複数回使用することができます。

次の例も考えてみましょう::

  from typing import TypeVar
  from collections.abc import Mapping

  T = TypeVar('T')

  class MyDict(Mapping[str, T]):
      ...

この場合、MyDict は単一のパラメータ T を持ちます。


抽象ジェネリック型
------------------------------------------------------------------------------------------

``Generic`` に使用されるメタクラスは ``abc.ABCMeta`` のサブクラスです。
ジェネリッククラスは抽象メソッドやプロパティを含めることで ABC になることができます。また、メタクラスの競合なしに ABC を基底クラスとして持つこともできます。

.. _`typevar-bound`:

上限を持つ型変数
------------------------------------------------------------------------------------------

型変数は ``TypeVar`` コンストラクタを使用する場合は ``bound=<type>`` を使用して、ジェネリック構文のネイティブ構文を使用する場合は ``: <type>`` を使用して上限を指定できます。 上限自体は型変数によってパラメータ化することはできません。 これは、型変数に対して明示的または暗黙的に置き換えられる実際の型が上限に :term:`assignable` でなければならないことを意味します。 例::

  from typing import TypeVar
  from collections.abc import Sized

  ST = TypeVar('ST', bound=Sized)

  def longer(x: ST, y: ST) -> ST:
      if len(x) > len(y):
          return x
      else:
          return y

  longer([1], [1, 2])  # OK、戻り値の型は list[int]
  longer({1}, {1, 2})  # OK、戻り値の型は set[int]
  longer([1], {1, 2})  # OK、戻り値の型は list[int] と set[int] のスーパータイプ

上限は型制約 (前述の ``AnyStr`` で使用されるもの) と組み合わせることはできません。 型制約は推論された型が制約型の正確な 1 つであることを要求しますが、上限は実際の型が上限に :term:`assignable` であることを要求するだけです。

.. _`variance`:

分散
------------------------------------------------------------------------------------------

``Employee`` クラスとそのサブクラス ``Manager`` を考えてみましょう。 さて、引数が ``list[Employee]`` とアノテートされた関数があるとします。 この関数を ``list[Manager]`` 型の変数を引数として呼び出すことは許可されるべきでしょうか? 多くの人は、結果を考慮せずに「はい、もちろん」と答えるでしょう。 しかし、関数についてもっと知っていない限り、型チェッカーはそのような呼び出しを拒否するべきです: 関数はリストに ``Employee`` インスタンスを追加するかもしれませんが、それは呼び出し元の変数の型に違反します。

このような引数は *反変* として機能しますが、関数が引数を変更しない場合に正しい直感的な答えは、引数が *共変* として機能することを要求します。 これらの概念のより長い紹介は `Wikipedia <https://en.wikipedia.org/wiki/Covariance_and_contravariance_%28computer_science%29>`_ および :pep:`483` にあります。ここでは型チェッカーの動作を制御する方法を示します。

古い ``TypeVar`` 構文を使用して宣言されたジェネリック型は、デフォルトですべての型変数で *不変* と見なされます。 これは、例えば ``list[Manager]`` が ``list[Employee]`` のスーパータイプでもサブタイプでもないことを意味します。

Python 3.12 以降の組み込みジェネリック構文を使用する場合の動作については、以下を参照してください。

共変または反変の型チェックが許容されるコンテナ型の宣言を容易にするために、型変数はキーワード引数 ``covariant=True`` または ``contravariant=True`` を受け入れます。 これらのうちの 1 つだけが渡されることができます。 そのような変数で定義されたジェネリック型は、対応する変数で共変または反変と見なされます。 慣例として、``covariant=True`` で定義された型変数には ``_co`` で終わる名前を使用し、``contravariant=True`` で定義された型変数には ``_contra`` で終わる名前を使用することが推奨されます。

典型的な例は、不変 (または読み取り専用) コンテナクラスを定義することです::

  from typing import TypeVar, Generic
  from collections.abc import Iterable, Iterator

  T_co = TypeVar('T_co', covariant=True)

  class ImmutableList(Generic[T_co]):
      def __init__(self, items: Iterable[T_co]) -> None: ...
      def __iter__(self) -> Iterator[T_co]: ...
      ...

  class Employee: ...

  class Manager(Employee): ...

  def dump_employees(emps: ImmutableList[Employee]) -> None:
      for emp in emps:
          ...

  mgrs: ImmutableList[Manager] = ImmutableList([Manager()])
  dump_employees(mgrs)  # OK

``typing`` の読み取り専用コレクションクラスはすべて、その型変数で共変と宣言されています (例: ``Mapping`` および ``Sequence``)。 可変コレクションクラス (例: ``MutableMapping`` および ``MutableSequence``) は不変と宣言されています。 反変型の例は ``Generator`` 型であり、``send()`` 引数型で反変です (以下参照)。

分散は、型変数がジェネリッククラスにバインドされている場合にのみ意味があります。 共変または反変として宣言された型変数がジェネリック関数または型エイリアスにバインドされている場合、型チェッカーはこれについてユーザーに警告することがあります。 ただし、その後の型解析では、そのような関数やエイリアスに関して宣言された分散を無視する必要があります::

  T = TypeVar('T', covariant=True)

  class A(Generic[T]):  # このコンテキストでは T は共変です
    ...

  def f(x: T) -> None:  # このコンテキストでは T の分散は意味がありません
    ...

  Alias = list[T] | set[T]  # このコンテキストでは T の分散は意味がありません

.. _`paramspec`:

ParamSpec
------------------------------------------------------------------------------------------

(元々 :pep:`612` によって指定されました。)

``ParamSpec`` 変数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

宣言
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

パラメータ仕様変数は、``typing.TypeVar`` で通常の型変数が定義されるのと同様に定義されます。

.. code-block::

   from typing import ParamSpec
   P = ParamSpec("P")         # 受け入れられます
   P = ParamSpec("WrongName") # 拒否されます。なぜなら P =/= WrongName だからです

ランタイムは、``bound``\ s および ``covariant`` および ``contravariant`` 引数を ``typing.TypeVar`` と同様に宣言で受け入れる必要がありますが、これらのオプションのセマンティクスの標準化は後の PEP に延期します。

.. _`paramspec_valid_use_locations`:

有効な使用場所
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

以前は、パラメータ引数のリスト (``[A, B, C]``) または省略記号 (「未定義のパラメータ」を示す) のみが ``typing.Callable`` の最初の「引数」として受け入れられていました。 ここでは、パラメータ仕様変数 (``Callable[P, int]``\ ) またはパラメータ仕様変数の連結 (``Callable[Concatenate[int, P], int]``\ ) を新たに追加します。

.. code-block::

   callable ::= Callable "[" parameters_expression, type_expression "]"

   parameters_expression ::=
     | "..."
     | "[" [ type_expression ("," type_expression)* ] "]"
     | parameter_specification_variable
     | concatenate "["
                      type_expression ("," type_expression)* ","
                      parameter_specification_variable
                   "]"

ここで ``parameter_specification_variable`` は ``typing.ParamSpec`` 変数であり、上記で定義された方法で宣言され、``concatenate`` は ``typing.Concatenate`` です。

以前と同様に、``parameters_expression``\ s 自体は型が期待される場所では受け入れられません

.. code-block::

   def foo(x: P) -> P: ...                           # 拒否されます
   def foo(x: Concatenate[int, P]) -> int: ...       # 拒否されます
   def foo(x: list[P]) -> None: ...                  # 拒否されます
   def foo(x: Callable[[int, str], P]) -> None: ...  # 拒否されます


ユーザー定義のジェネリッククラス
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

``Generic[T]`` を継承することでクラスをジェネリックにするのと同様に、``Generic[P]`` を継承することでクラスをパラメータ仕様変数でジェネリックにすることができます。

.. code-block::

   T = TypeVar("T")
   P_2 = ParamSpec("P_2")

   class X(Generic[T, P]):
     f: Callable[P, int]
     x: T

   def f(x: X[int, P_2]) -> str: ...                    # 受け入れられます
   def f(x: X[int, Concatenate[int, P_2]]) -> str: ...  # 受け入れられます
   def f(x: X[int, [int, bool]]) -> str: ...            # 受け入れられます
   def f(x: X[int, ...]) -> str: ...                    # 受け入れられます
   def f(x: X[int, int]) -> str: ...                    # 拒否されます

または、Python 3.12 以降のジェネリックの組み込み構文を使用して同等に::

  class X[T, **P]:
    f: Callable[P, int]
    x: T

上記のルールに従って、単一の ``ParamSpec`` に関してジェネリックなクラスの具体的なインスタンスをスペルするには、見苦しい二重角括弧を省略することができます。

.. code-block::

   class Z(Generic[P]):
     f: Callable[P, int]

   def f(x: Z[[int, str, bool]]) -> str: ...   # 受け入れられます
   def f(x: Z[int, str, bool]) -> str: ...     # 同等

   # Z[[int, str, bool]] と Z[int, str, bool] の両方がこれを表します:
   class Z_instantiated:
     f: Callable[[int, str, bool], int]

セマンティクス
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

シグネチャに ``ParamSpec`` 変数を含む関数呼び出しの戻り値の型の推論ルールは、``TypeVar``\ s を含むものの評価に関するルールと類似しています。

.. code-block::

   def changes_return_type_to_str(x: Callable[P, int]) -> Callable[P, str]: ...

   def returns_int(a: str, b: bool) -> int: ...

   f = changes_return_type_to_str(returns_int) # f の型は次のようになります:
                                               # (a: str, b: bool) -> str

   f("A", True)               # 受け入れられます
   f(a="A", b=True)           # 受け入れられます
   f("A", "A")                # 拒否されます

   expects_str(f("A", True))  # 受け入れられます
   expects_int(f("A", True))  # 拒否されます

従来の ``TypeVars``\ と同様に、ユーザーは同じ ``ParamSpec`` を関数の引数に複数回含めることができ、複数の引数間の依存関係を示すことができます。 これらの場合、型チェッカーは共通の動作上のスーパータイプ (つまり、すべての有効な呼び出しが両方のサブタイプで有効なパラメータのセット) を解決することを選択できますが、そうする義務はありません。

.. code-block::

   P = ParamSpec("P")

   def foo(x: Callable[P, int], y: Callable[P, int]) -> Callable[P, bool]: ...

   def x_y(x: int, y: str) -> int: ...
   def y_x(y: int, x: str) -> int: ...

   foo(x_y, x_y)  # (x: int, y: str) -> bool を返すべきです
                  # (2 つの位置またはキーワード引数を持つ呼び出し可能なもの)

   foo(x_y, y_x)  # (a: int, b: str, /) -> bool を返すことができます
                  # (2 つの位置専用パラメータを持つ呼び出し可能なもの)
                  # これは、両方の呼び出し可能なものの型が Callable[[int, str], int] の動作上のサブタイプであるためです


   def keyword_only_x(*, x: int) -> int: ...
   def keyword_only_y(*, y: int) -> int: ...
   foo(keyword_only_x, keyword_only_y) # 拒否されます

``ParamSpec``\ s でジェネリックなユーザー定義クラスのコンストラクタは同様に評価されるべきです。

.. code-block::

   U = TypeVar("U")

   class Y(Generic[U, P]):
     f: Callable[P, str]
     prop: U

     def __init__(self, f: Callable[P, str], prop: U) -> None:
       self.f = f
       self.prop = prop

   def a(q: int) -> str: ...

   Y(a, 1)   # Y[int, (q: int)] に解決されるべきです
   Y(a, 1).f # (q: int) -> str に解決されるべきです

``Concatenate[X, Y, P]`` のセマンティクスは、``P`` で表されるパラメータに 2 つの位置専用パラメータが前置されることを表します。 これにより、有限数のパラメータを追加、削除、または変換する高階関数を表すことができます。

.. code-block::

   def bar(x: int, *args: bool) -> int: ...

   def add(x: Callable[P, int]) -> Callable[Concatenate[str, P], bool]: ...

   add(bar)       # (a: str, /, x: int, *args: bool) -> bool を返すべきです

   def remove(x: Callable[Concatenate[int, P], int]) -> Callable[P, bool]: ...

   remove(bar)    # (*args: bool) -> bool を返すべきです

   def transform(
     x: Callable[Concatenate[int, P], int]
   ) -> Callable[Concatenate[str, P], bool]: ...

   transform(bar) # (a: str, /, *args: bool) -> bool を返すべきです

これにより、``R`` を返す関数は ``typing.Callable[P, R]`` を満たすことができますが、最初の位置で ``X`` を持つ位置専用で呼び出すことができる関数のみが ``typing.Callable[Concatenate[X, P], R]`` を満たすことができます。

.. code-block::

   def expects_int_first(x: Callable[Concatenate[int, P], int]) -> None: ...

   @expects_int_first # 拒否されます
   def one(x: str) -> int: ...

   @expects_int_first # 拒否されます
   def two(*, x: int) -> int: ...

   @expects_int_first # 拒否されます
   def three(**kwargs: int) -> int: ...

   @expects_int_first # 受け入れられます
   def four(*args: int) -> int: ...

これらの機能を使用しても、まだサポートされていないデコレータのクラスがあります:

* **可変** 数のパラメータを追加、削除、変更するもの (例えば、 ``functools.partial`` は ``ParamSpec`` を使用しても型付けできません)
* キーワード専用パラメータを追加、削除、変更するもの。

``ParamSpec`` のコンポーネント
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``ParamSpec`` は位置およびキーワードでアクセス可能なパラメータの両方をキャプチャしますが、残念ながらランタイムにはこれらの両方をキャプチャするオブジェクトは存在しません。 代わりに、これらを ``*args`` と ``**kwargs`` に分割する必要があります。 これにより、単一の ``ParamSpec`` をこれら 2 つのコンポーネントに分割し、それらを呼び出しに戻すことができます。 これを行うために、``P.args`` を呼び出しの位置引数のタプルとして表し、``P.kwargs`` をキーワードと値の対応する ``Mapping`` として表します。

有効な使用場所
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

これらの「プロパティ」は、``*args`` および ``**kwargs`` のアノテートされた型としてのみ使用でき、すでにスコープ内にある ParamSpec からアクセスされます。

.. code-block::

   def puts_p_into_scope(f: Callable[P, int]) -> None:

     def inner(*args: P.args, **kwargs: P.kwargs) -> None:      # 受け入れられます
       pass

     def mixed_up(*args: P.kwargs, **kwargs: P.args) -> None:   # 拒否されます
       pass

     def misplaced(x: P.args) -> None:                          # 拒否されます
       pass

   def out_of_scope(*args: P.args, **kwargs: P.kwargs) -> None: # 拒否されます
     pass


さらに、Python のデフォルトのパラメータの種類 (\ ``(x: int)``\ ) は位置および名前を通じてアドレス指定できるため、``(*args: P.args, **kwargs: P.kwargs)`` 関数の 2 つの有効な呼び出しは同じパラメータの異なるパーティションを与える可能性があります。 したがって、これらの特別な型が一緒に導入され、一緒に使用されることを確認する必要があります。これにより、すべての可能なパーティションに対して使用が有効になります。

.. code-block::

   def puts_p_into_scope(f: Callable[P, int]) -> None:

     stored_args: P.args                           # 拒否されます

     stored_kwargs: P.kwargs                       # 拒否されます

     def just_args(*args: P.args) -> None:         # 拒否されます
       pass

     def just_kwargs(**kwargs: P.kwargs) -> None:  # 拒否されます
       pass


セマンティクス
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

これらの要件が満たされると、次のような特性を活用できます:


* 関数内では、``args`` の型は ``P.args`` であり、通常のアノテーションのように ``tuple[P.args, ...]`` ではありません
  (``**kwargs`` も同様です)

  * この特別なケースは、与えられた呼び出しの ``args``/``kwargs`` の異種内容をカプセル化するために必要であり、無期限のタプル/辞書型では表現できません。

* ``Callable[P, R]`` 型の関数は、``args`` の型が ``P.args`` であり、``kwargs`` の型が ``P.kwargs`` であり、それらの型が同じ関数宣言から派生している場合にのみ ``(*args, **kwargs)`` で呼び出すことができます。
* ``def inner(*args: P.args, **kwargs: P.kwargs) -> X`` と宣言された関数は ``Callable[P, X]`` 型を持ちます。

これらの 3 つの特性により、パラメータを保持するデコレータを完全に型チェックする能力が得られます。

.. code-block::

   def decorator(f: Callable[P, int]) -> Callable[P, None]:

     def foo(*args: P.args, **kwargs: P.kwargs) -> None:

       f(*args, **kwargs)    # 受け入れられます。int に解決されるべきです

       f(*kwargs, **args)    # 拒否されます

       f(1, *args, **kwargs) # 拒否されます

     return foo              # 受け入れられます

これを ``Concatenate`` に拡張するには、次のプロパティを宣言します:

* ``Callable[Concatenate[A, B, P], R]`` 型の関数は、``args`` および ``kwargs`` が ``P`` の対応するコンポーネントである場合にのみ ``(a, b, *args, **kwargs)`` で呼び出すことができます。 ``a`` は ``A`` 型であり、``b`` は ``B`` 型です。
* ``def inner(a: A, b: B, *args: P.args, **kwargs: P.kwargs) -> R`` と宣言された関数は ``Callable[Concatenate[A, B, P], R]`` 型を持ちます。 ``*args`` と ``**kwargs`` の間にキーワード専用パラメータを配置することは禁止されています。

.. code-block::

   def add(f: Callable[P, int]) -> Callable[Concatenate[str, P], None]:

     def foo(s: str, *args: P.args, **kwargs: P.kwargs) -> None:  # 受け入れられます
       pass

     def bar(*args: P.args, s: str, **kwargs: P.kwargs) -> None:  # 拒否されます
       pass

     return foo                                                   # 受け入れられます


   def remove(f: Callable[Concatenate[int, P], int]) -> Callable[P, None]:

     def foo(*args: P.args, **kwargs: P.kwargs) -> None:
       f(1, *args, **kwargs) # 受け入れられます

       f(*args, 1, **kwargs) # 拒否されます

       f(*args, **kwargs)    # 拒否されます

     return foo

パラメータの名前は ``ParamSpec`` コンポーネントの前にあるため、結果の ``Concatenate`` では言及されません。 これは、これらのパラメータが名前付き引数を介してアドレス指定できないことを意味します:

.. code-block::

   def outer(f: Callable[P, None]) -> Callable[P, None]:
     def foo(x: int, *args: P.args, **kwargs: P.kwargs) -> None:
       f(*args, **kwargs)

     def bar(*args: P.args, **kwargs: P.kwargs) -> None:
       foo(1, *args, **kwargs)   # 受け入れられます
       foo(x=1, *args, **kwargs) # 拒否されます

     return bar

.. _above:

これは実装の便宜ではなく、健全性の要件です。 もし 2 番目の呼び出しスタイルを許可すると、次のスニペットは問題になります。

.. code-block::

   @outer
   def problem(*, x: object) -> None:
     pass

   problem(x="uh-oh")

``bar`` 内では、``TypeError: foo() got multiple values for argument 'x'`` というエラーが発生します。 これらの連結された引数を位置指定でアドレス指定することを要求することで、この種の問題を回避し、これらの型をスペルするための構文を簡素化します。 これが、``(*args: P.args, s: str, **kwargs: P.kwargs)`` の形式のシグネチャを拒否する必要がある理由でもあります。

これらの前置された位置パラメータの 1 つに自由な ``ParamSpec`` が含まれている場合、その ``ParamSpec`` のコンポーネントを抽出する目的でその変数をスコープ内と見なします。 これにより、次のような型をスペルすることができます:

.. code-block::

   def twice(f: Callable[P, int], *args: P.args, **kwargs: P.kwargs) -> int:
     return f(*args, **kwargs) + f(*args, **kwargs)

上記の例の ``twice`` の型は ``Callable[Concatenate[Callable[P, int], P], int]`` であり、``P`` は外部の ``Callable`` によってバインドされます。 これには次のセマンティクスがあります:

.. code-block::

   def a_int_b_str(a: int, b: str) -> int:
     pass

   twice(a_int_b_str, 1, "A")       # 受け入れられます

   twice(a_int_b_str, b="A", a=1)   # 受け入れられます

   twice(a_int_b_str, "A", 1)       # 拒否されます

.. _`typevartuple`:

TypeVarTuple
------------------------------------------------------------------------------------------

(元々 :pep:`646` によって指定されました。)

``TypeVarTuple`` は単一の型ではなく、*タプル* 型のプレースホルダーとして機能します。

さらに、星演算子を使用して ``TypeVarTuple`` インスタンスや ``tuple[int, str]`` などのタプル型を「アンパック」する新しい使用法を導入します。 ``TypeVarTuple`` またはタプル型をアンパックすることは、変数や値のタプルをアンパックすることの型付けの等価物です。

型変数タプル
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

通常の型変数が ``int`` などの単一の型の代わりに使用されるのと同様に、型変数 *タプル* は ``tuple[int, str]`` などの *タプル* 型の代わりに使用されます。

型変数タプルは次のように作成および使用されます:

::

    from typing import TypeVarTuple

    Ts = TypeVarTuple('Ts')

    class Array(Generic[*Ts]):
      ...

    def foo(*args: *Ts):
      ...

または、Python 3.12 以降のジェネリックの組み込み構文を使用して::

    class Array[*Ts]:
      ...

    def foo[*Ts](*args: *Ts):
      ...

ジェネリッククラスでの型変数タプルの使用
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型変数タプルは、``tuple`` にパックされた複数の個別の型変数のように動作します。 これを理解するために、次の例を考えてみましょう:

::

  Shape = TypeVarTuple('Shape')

  class Array(Generic[*Shape]): ...

  Height = NewType('Height', int)
  Width = NewType('Width', int)
  x: Array[Height, Width] = Array()

ここでの ``Shape`` 型変数タプルは、``tuple[T1, T2]`` のように動作し、``T1`` と ``T2`` は型変数です。 これらの型変数を ``Array`` の型パラメータとして使用するには、星演算子を使用して型変数タプルを *アンパック* する必要があります: ``*Shape``。 その後、``Array`` のシグネチャは単に ``class Array(Generic[T1, T2]): ...`` と書いた場合と同じように動作します。

ただし、``Generic[T1, T2]`` とは異なり、``Generic[*Shape]`` を使用すると、任意の数の型パラメータでクラスをパラメータ化できます。 つまり、``Array[Height, Width]`` のようなランク 2 の配列を定義するだけでなく、ランク 3 の配列、ランク 4 の配列なども定義できます:

::

  Time = NewType('Time', int)
  Batch = NewType('Batch', int)
  y: Array[Batch, Height, Width] = Array()
  z: Array[Time, Batch, Height, Width] = Array()

関数での型変数タプルの使用
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型変数タプルは通常の ``TypeVar`` と同様に使用できます。
これには、上記のようなクラス定義だけでなく、関数シグネチャや変数アノテーションも含まれます:

::

    class Array(Generic[*Shape]):

        def __init__(self, shape: tuple[*Shape]):
            self._shape: tuple[*Shape] = shape

        def get_shape(self) -> tuple[*Shape]:
            return self._shape

    shape = (Height(480), Width(640))
    x: Array[Height, Width] = Array(shape)
    y = abs(x)  # 推論された型は Array[Height, Width]
    z = x + x   # 推論された型は Array[Height, Width]

型変数タプルは常にアンパックされる必要があります
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

前の例では、``__init__`` の ``shape`` 引数が ``tuple[*Shape]`` とアノテートされていたことに注意してください。 なぜこれが必要なのでしょうか - ``Shape`` が ``tuple[T1, T2, ...]`` のように動作する場合、``shape`` 引数を直接 ``Shape`` とアノテートすることはできないのでしょうか?

実際には、型変数タプルは常に *アンパック* された形式 (つまり、星演算子が付いた形式) で使用される必要があります。 これには 2 つの理由があります:

* 型変数タプルをパックまたはアンパック形式で使用するかどうかについての混乱を避けるため (「うーん、'``-> Shape``' と書くべきか、'``-> tuple[Shape]``' と書くべきか、'``-> tuple[*Shape]``' と書くべきか...」)
* 読みやすさを向上させるため: 星演算子は、型変数タプルが通常の型変数ではないことを明示的に示す視覚的な指標としても機能します。

分散、型制約、型境界: まだサポートされていません
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

``TypeVarTuple`` はまだ次の仕様をサポートしていません:

* 分散 (例: ``TypeVar('T', covariant=True)``)
* 型制約 (``TypeVar('T', int, float)``)
* 型境界 (``TypeVar('T', bound=ParentClass)``)

これらの引数の動作については、可変長ジェネリックがフィールドでテストされた後の将来の PEP に委ねます。 PEP 646 の時点では、型変数タプルは不変です。

型変数タプルの等価性
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

同じ ``TypeVarTuple`` インスタンスがシグネチャやクラスの複数の場所で使用される場合、有効な型推論は ``TypeVarTuple`` を型のタプルにバインドすることです:

::

  def foo(arg1: tuple[*Ts], arg2: tuple[*Ts]): ...

  a = (0,)
  b = ('0',)
  foo(a, b)  # Ts を tuple[int | str] にバインドできますか?

これを許可しません。型の結合はタプル内に現れることはできません。
型変数タプルがシグネチャの複数の場所に現れる場合、型は正確に一致する必要があります (型パラメータのリストは同じ長さであり、型パラメータ自体も同一である必要があります):

::

  def pointwise_multiply(
      x: Array[*Shape],
      y: Array[*Shape]
  ) -> Array[*Shape]: ...

  x: Array[Height]
  y: Array[Width]
  z: Array[Height, Width]
  pointwise_multiply(x, x)  # 有効
  pointwise_multiply(x, y)  # エラー
  pointwise_multiply(x, z)  # エラー

複数の型変数タプル: 許可されていません
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型パラメータリストに複数の型変数タプルを含めることはできません:

::

    class Array(Generic[*Ts1, *Ts2]): ...  # エラー

理由は、複数の型変数タプルがあると、どのパラメータがどの型変数タプルにバインドされるかが曖昧になるためです: ::

    x: Array[int, str, bool]  # Ts1 = ???, Ts2 = ???

型の連結
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

型変数タプルは単独で存在する必要はありません。通常の型はプレフィックスおよび/またはサフィックスとして使用できます:

::

    Shape = TypeVarTuple('Shape')
    Batch = NewType('Batch', int)
    Channels = NewType('Channels', int)

    def add_batch_axis(x: Array[*Shape]) -> Array[Batch, *Shape]: ...
    def del_batch_axis(x: Array[Batch, *Shape]) -> Array[*Shape]: ...
    def add_batch_channels(
      x: Array[*Shape]
    ) -> Array[Batch, *Shape, Channels]: ...

    a: Array[Height, Width]
    b = add_batch_axis(a)      # 推論された型は Array[Batch, Height, Width]
    c = del_batch_axis(b)      # Array[Height, Width]
    d = add_batch_channels(a)  # Array[Batch, Height, Width, Channels]


通常の ``TypeVar`` インスタンスもプレフィックスおよび/またはサフィックスとして使用できます:

::

    T = TypeVar('T')
    Ts = TypeVarTuple('Ts')

    def prefix_tuple(
        x: T,
        y: tuple[*Ts]
    ) -> tuple[T, *Ts]: ...

    z = prefix_tuple(x=0, y=(True, 'a'))
    # 推論された型は tuple[int, bool, str]

タプル型のアンパック
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

型変数タプルをアンパックできることを述べました。
一貫性のために、タプル型もアンパックすることを許可します。 これにより、いくつかの興味深い機能が有効になります。


無制限のタプル型のアンパック
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

無制限のタプルをアンパックすることは、正確な要素を気にせず、不要な ``TypeVarTuple`` を定義したくない関数シグネチャで役立ちます:

::

    def process_batch_channels(
        x: Array[Batch, *tuple[Any, ...], Channels]
    ) -> None:
        ...


    x: Array[Batch, Height, Width, Channels]
    process_batch_channels(x)  # OK
    y: Array[Batch, Channels]
    process_batch_channels(y)  # OK
    z: Array[Batch]
    process_batch_channels(z)  # エラー: Channels が必要です。


``*tuple[int, ...]`` を ``*Ts`` が期待される場所に渡すこともできます。 これは、特に動的なコードがあり、次元の数や各次元の正確な型を指定できない場合に役立ちます。 そのような場合、無制限のタプルにスムーズにフォールバックできます:

::

    y: Array[*tuple[Any, ...]] = read_from_file()

    def expect_variadic_array(
        x: Array[Batch, *Shape]
    ) -> None: ...

    expect_variadic_array(y)  # OK

    def expect_precise_array(
        x: Array[Batch, Height, Width, Channels]
    ) -> None: ...

    expect_precise_array(y)  # OK

``Array[*tuple[Any, ...]]`` は任意の数の次元を持つ配列を表します。 これは、``expect_variadic_array`` の呼び出しで ``Batch`` が ``Any`` にバインドされ、``Shape`` が ``tuple[Any, ...]`` にバインドされることを意味します。 ``expect_precise_array`` の呼び出しでは、``Batch``、``Height``、``Width``、および ``Channels`` の変数はすべて ``Any`` にバインドされます。

これにより、動的なコードを安全でないことを明示的にマークしながら (``y: Array[*tuple[Any, ...]]`` を使用して) 優雅に処理できます。 そうしないと、型チェッカーからのノイズの多いエラーが発生し、変数 ``y`` を使用するたびに妨げられ、コードベースを ``TypeVarTuple`` を使用するように移行する際に妨げられます。

.. _args_as_typevartuple:

``*args`` を型変数タプルとして使用する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:ref:`この仕様 <annotating-args-kwargs>` では、``*args`` に型アノテーションが提供されると、すべての引数はアノテートされた型でなければならないと述べています。 つまり、``*args`` を ``int`` 型として指定すると、*すべての* 引数は ``int`` 型でなければなりません。 これにより、異種引数型を取る関数の型シグネチャを指定する能力が制限されます。

ただし、``*args`` が型変数タプルとしてアノテートされている場合、個々の引数の型は型変数タプル内の型になります:

::

    Ts = TypeVarTuple('Ts')

    def args_to_tuple(*args: *Ts) -> tuple[*Ts]: ...

    args_to_tuple(1, 'a')  # 推論された型は tuple[int, str]

上記の例では、``Ts`` は ``tuple[int, str]`` にバインドされています。 引数が渡されない場合、型変数タプルは空のタプル ``tuple[()]`` のように動作します。

通常通り、任意のタプル型をアンパックできます。 例えば、他の型のタプル内で型変数タプルを使用することで、可変長引数リストのプレフィックスやサフィックスを参照できます。 例えば:

::

    # os.execle は引数 'path, arg0, arg1, ..., env' を取ります
    def execle(path: str, *args: *tuple[*Ts, Env]) -> None: ...

これは次のものとは異なります:

::

    def execle(path: str, *args: *Ts, env: Env) -> None: ...

これは ``env`` をキーワード専用引数にします。

アンパックされた無制限のタプルを使用することは、``*args: int`` の動作と同等です。これは、ゼロまたはそれ以上の ``int`` 型の値を受け入れます:

::

    def foo(*args: *tuple[int, ...]) -> None: ...

    # 同等:
    def foo(*args: int) -> None: ...

タプル型のアンパックにより、異種 ``*args`` のより正確な型も指定できます。 次の関数は、最初に ``int`` を取り、ゼロまたはそれ以上の ``str`` 値を取り、最後に ``str`` を取ります:

::

    def foo(*args: *tuple[int, *tuple[str, ...], str]) -> None: ...

完全性のために、具体的なタプルをアンパックすることで、固定数の異種型の ``*args`` を指定できることを述べておきます:

::

    def foo(*args: *tuple[int, str]) -> None: ...

    foo(1, "hello")  # OK

通常の型変数とは異なり、型変数タプルインスタンスをそのまま ``*args`` の型としてアノテートすることは許可されていません:

::

    def foo(*args: Ts): ...  # 無効

``*args`` は ``*Ts`` を直接アノテートできる唯一のケースです。他の引数は ``*Ts`` を使用して何か他のものをパラメータ化する必要があります。 例: ``tuple[*Ts]``。
``*args`` 自体が ``tuple[*Ts]`` としてアノテートされている場合、古い動作が適用されます: すべての引数は同じ型でパラメータ化された ``tuple`` でなければなりません。

::

    def foo(*args: tuple[*Ts]): ...

    foo((0,), (1,))    # 有効
    foo((0,), (1, 2))  # エラー
    foo((0,), ('1',))  # エラー

最後に、型変数タプルを ``**kwargs`` の型として使用することはできません。 (この機能の使用例はまだ知られていないため、将来の PEP のためにこの領域を新鮮なままにしておくことを好みます。)

::

    # 無効
    def foo(**kwargs: *Ts): ...

型変数タプルを含む ``Callable``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

型変数タプルは ``Callable`` の引数セクションでも使用できます:

::

    class Process:
      def __init__(
        self,
        target: Callable[[*Ts], None],
        args: tuple[*Ts],
      ) -> None: ...

    def func(arg1: int, arg2: str) -> None: ...

    Process(target=func, args=(0, 'foo'))  # 有効
    Process(target=func, args=('foo', 0))  # エラー

他の型や通常の型変数も型変数タプルにプレフィックス/サフィックスとして使用できます:

::

    T = TypeVar('T')

    def foo(f: Callable[[int, *Ts, T], tuple[T, *Ts]]): ...

アンパックされた項目を含む ``Callable`` の動作は、項目が ``TypeVarTuple`` であろうとタプル型であろうと、要素を ``*args`` の型として扱うことです。 したがって、``Callable[[*Ts], None]`` は次の関数の型として扱われます:

::

    def foo(*args: *Ts) -> None: ...

``Callable[[int, *Ts, T], tuple[T, *Ts]]`` は次の関数の型として扱われます:

::

    def foo(*args: *tuple[int, *Ts, T]) -> tuple[T, *Ts]: ...

型パラメータが指定されていない場合の動作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

型変数タプルにデフォルト値がないジェネリッククラスが型パラメータなしで使用される場合、型変数タプルは ``tuple[Any, ...]`` に置き換えられたかのように動作します:

::

    def takes_any_array(arr: Array): ...

    # 同等:
    def takes_any_array(arr: Array[*tuple[Any, ...]]): ...

    x: Array[Height, Width]
    takes_any_array(x)  # 有効
    y: Array[Time, Height, Width]
    takes_any_array(y)  # これも有効

これにより、段階的な型付けが可能になります: 例えば、TensorFlow の ``Tensor`` を受け入れる既存の関数は、ライブラリが更新されて ``Tensor`` がジェネリックになり、呼び出しコードが ``Tensor[Height, Width]`` を渡す場合でも有効です。

逆の方向でも動作します:

::

    def takes_specific_array(arr: Array[Height, Width]): ...

    z: Array
    # 実際には Array[*tuple[Any, ...]]

    takes_specific_array(z)

(詳細については、`無制限のタプル型のアンパック`_ セクションを参照してください。)

このようにして、ライブラリが ``Array[Height, Width]`` のような型を使用するように更新されても、ユーザーはコード全体に型アノテーションを適用することを強制されません。 ユーザーは依然としてコードのどの部分に型を付け、どの部分に型を付けないかを選択できます。

エイリアス
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ジェネリックエイリアスは、通常の型変数と同様に型変数タプルを使用して作成できます:

::

    IntTuple = tuple[int, *Ts]
    NamedArray = tuple[str, Array[*Ts]]

    IntTuple[float, bool]  # tuple[int, float, bool] と同等
    NamedArray[Height]     # tuple[str, Array[Height]] と同等

この例が示すように、エイリアスに渡されるすべての型パラメータは型変数タプルにバインドされます。

これにより、固定形状やデータ型の配列の便利なエイリアスを定義できます:

::

    Shape = TypeVarTuple('Shape')
    DType = TypeVar('DType')
    class Array(Generic[DType, *Shape]):

    # 例: Float32Array[Height, Width, Channels]
    Float32Array = Array[np.float32, *Shape]

    # 例: Array1D[np.uint8]
    Array1D = Array[DType, Any]

明示的に空の型パラメータリストが指定された場合、エイリアスの型変数タプルは空に設定されます:

::

    IntTuple[()]    # tuple[int] と同等
    NamedArray[()]  # tuple[str, Array[()]] と同等

型パラメータリストが完全に省略された場合、指定されていない型変数タプルは ``tuple[Any, ...]`` として扱われます ( `型パラメータが指定されていない場合の動作`_ に類似):

::

    def takes_float_array_of_any_shape(x: Float32Array): ...
    x: Float32Array[Height, Width] = Array()
    takes_float_array_of_any_shape(x)  # 有効

    def takes_float_array_with_specific_shape(
        y: Float32Array[Height, Width]
    ): ...
    y: Float32Array = Array()
    takes_float_array_with_specific_shape(y)  # 有効

通常の ``TypeVar`` インスタンスもそのようなエイリアスで使用できます:

::

    T = TypeVar('T')
    Foo = tuple[T, *Ts]

    # T は str にバインドされ、Ts は tuple[int] にバインドされます
    Foo[str, int]
    # T は float にバインドされ、Ts は tuple[()] にバインドされます
    Foo[float]
    # T は Any にバインドされ、Ts は tuple[Any, ...] にバインドされます
    Foo


エイリアスでの置換
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

前のセクションでは、型引数が単純な型である場合のジェネリックエイリアスの単純な使用についてのみ説明しました。 ただし、いくつかのよりエキゾチックな構成も可能です。


型引数は可変長である可能性があります
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

まず、ジェネリックエイリアスへの型引数は可変長である可能性があります。 例えば、``TypeVarTuple`` を型引数として使用できます:

::

    Ts1 = TypeVarTuple('Ts1')
    Ts2 = TypeVarTuple('Ts2')

    IntTuple = tuple[int, *Ts1]
    IntFloatTuple = IntTuple[float, *Ts2]  # 有効

ここで、``IntTuple`` エイリアスの ``*Ts1`` は ``tuple[float, *Ts2]`` にバインドされ、``IntFloatTuple`` エイリアスは ``tuple[int, float, *Ts2]`` と同等になります。

アンパックされた任意の長さのタプルも型引数として使用できます。同様の効果があります:

::

    IntFloatsTuple = IntTuple[*tuple[float, ...]]  # 有効

ここで、``*Ts1`` は ``*tuple[float, ...]`` にバインドされ、``IntFloatsTuple`` は ``tuple[int, *tuple[float, ...]]`` と同等になります: ``int`` の後にゼロまたはそれ以上の ``float``\ s が続くタプルです。


可変長引数には可変長エイリアスが必要です
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

可変長型引数は、それ自体が可変長のジェネリックエイリアスでのみ使用できます。 例えば:

::

    T = TypeVar('T')

    IntTuple = tuple[int, T]

    IntTuple[str]                 # 有効
    IntTuple[*Ts]                 # 無効
    IntTuple[*tuple[float, ...]]  # 無効

ここで、``IntTuple`` は単一の型引数を取る *非* 可変長のジェネリックエイリアスです。 したがって、任意の数の型を表す ``*Ts`` や ``*tuple[float, ...]`` を型引数として受け入れることはできません。


型変数と型変数タプルの両方を持つエイリアス
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

`エイリアス`_ では、エイリアスが ``TypeVar``\ s と ``TypeVarTuple``\ s の両方でジェネリックである可能性があることを簡単に述べました:

::

    T = TypeVar('T')
    Foo = tuple[T, *Ts]

    Foo[str, int]         # T は str にバインドされ、Ts は tuple[int] にバインドされます
    Foo[str, int, float]  # T は str にバインドされ、Ts は tuple[int, float] にバインドされます

`複数の型変数タプル: 許可されていません`_ に従って、エイリアスの型パラメータには最大 1 つの ``TypeVarTuple`` を含めることができます。 ただし、``TypeVarTuple`` は前後に任意の数の ``TypeVar``\ s と組み合わせることができます:

::

    T1 = TypeVar('T1')
    T2 = TypeVar('T2')
    T3 = TypeVar('T3')

    tuple[*Ts, T1, T2]      # 有効
    tuple[T1, T2, *Ts]      # 有効
    tuple[T1, *Ts, T2, T3]  # 有効

これらの型変数を提供された型引数に置換するために、型パラメータリストの先頭または末尾の型変数は最初に型引数を消費し、残りの型引数は ``TypeVarTuple`` にバインドされます:

::

    Shrubbery = tuple[*Ts, T1, T2]

    Shrubbery[str, bool]              # T2=bool、T1=str、Ts=tuple[()]
    Shrubbery[str, bool, float]       # T2=float、T1=bool、Ts=tuple[str]
    Shrubbery[str, bool, float, int]  # T2=int、T1=float、Ts=tuple[str, bool]

    Ptang = tuple[T1, *Ts, T2, T3]

    Ptang[str, bool, float]       # T1=str、T3=float、T2=bool、Ts=tuple[()]
    Ptang[str, bool, float, int]  # T1=str、T3=int、T2=float、Ts=tuple[bool]

このような場合の最小の型引数の数は、型変数の数によって設定されます:

::

    Shrubbery[int]  # 無効: Shrubbery には少なくとも 2 つの型引数が必要です


任意の長さのタプルの分割
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

アンパックされた任意の長さのタプルが型変数タプルを含むエイリアスへの型引数として使用される場合、最後の複雑なケースが発生します:

::

    Elderberries = tuple[*Ts, T1]
    Hamster = Elderberries[*tuple[int, ...]]  # 有効

このような場合、任意の長さのタプルは型変数タプルと型変数の間で分割されます。 任意の長さのタプルには型変数の数と同じ数の項目が含まれていると仮定し、内部型 (ここでは ``int``) の個別のインスタンスが存在する型変数にバインドされます。 任意の長さのタプルの「残り」 (ここでは ``*tuple[int, ...]``) は型変数タプルにバインドされます。

ここでは、``Hamster`` は ``tuple[*tuple[int, ...], int]`` と同等です: ゼロまたはそれ以上の ``int``\ s の後に ``int`` が続くタプルです。

もちろん、そのような分割は必要な場合にのみ発生します。 例えば、次のようにした場合:

::

   Elderberries[*tuple[int, ...], str]

分割は発生しません。 ``T1`` は ``str`` にバインドされ、``Ts`` は ``*tuple[int, ...]`` にバインドされます。

特に厄介なケースでは、``TypeVarTuple`` は型と任意の長さのタプル型の両方を消費することがあります:

::

    Elderberries[str, *tuple[int, ...]]

ここでは、``T1`` は ``int`` にバインドされ、``Ts`` は ``tuple[str, *tuple[int, ...]]`` にバインドされます。 この式は、``tuple[str, *tuple[int, ...], int]`` と同等です: ``str`` の後にゼロまたはそれ以上の ``int``\ s が続き、最後に ``int`` が続くタプルです。


型変数タプルは分割できません
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

最後に、型変数タプルが型変数や型変数タプルの引数リストに含まれている場合、任意の長さのタプルは型変数や型変数タプルの引数リストに分割されることがありますが、同じことは型変数タプルには当てはまりません:

::

    Ts1 = TypeVarTuple('Ts1')
    Ts2 = TypeVarTuple('Ts2')

    Camelot = tuple[T, *Ts1]
    Camelot[*Ts2]  # 無効

これは、アンパックされた任意の長さのタプルの場合とは異なり、``TypeVarTuple`` の内部を「覗き見る」方法がないためです。


個々の型にアクセスするためのオーバーロード
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

型変数タプルの個々の型にアクセスする必要がある場合、オーバーロードを使用して型変数タプルの代わりに個々の ``TypeVar`` インスタンスを使用できます:

::

    Shape = TypeVarTuple('Shape')
    Axis1 = TypeVar('Axis1')
    Axis2 = TypeVar('Axis2')
    Axis3 = TypeVar('Axis3')

    class Array(Generic[*Shape]):

      @overload
      def transpose(
        self: Array[Axis1, Axis2]
      ) -> Array[Axis2, Axis1]: ...

      @overload
      def transpose(
        self: Array[Axis1, Axis2, Axis3]
      ) -> Array[Axis3, Axis2, Axis1]: ...

(特に配列形状操作の場合、可能なランクごとにオーバーロードを指定することは非常に面倒な解決策です。 ただし、追加の型操作メカニズムがない場合、これが最善の方法です。)

.. _`type_parameter_defaults`:

型パラメータのデフォルト
------------------------------------------------------------------------------------------

(元々 :pep:`696` によって指定されました。)

型変数、ParamSpec、または TypeVarTuple にデフォルト値を提供できます。

デフォルトの順序とサブスクリプションルール
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

デフォルトの順序は標準の関数パラメータルールに従う必要があり、デフォルト値を持たない型パラメータはデフォルト値を持つ型パラメータの後に続くことはできません。 これを行うと、ランタイムで ``TypeError`` が発生する可能性があり、型チェッカーはこれをエラーとしてフラグを立てる必要があります。

::

   DefaultStrT = TypeVar("DefaultStrT", default=str)
   DefaultIntT = TypeVar("DefaultIntT", default=int)
   DefaultBoolT = TypeVar("DefaultBoolT", default=bool)
   T = TypeVar("T")
   T2 = TypeVar("T2")

   class NonDefaultFollowsDefault(Generic[DefaultStrT, T]): ...  # 無効: デフォルトを持つ型変数の後に非デフォルトの型変数が続くことはできません


   class NoNonDefaults(Generic[DefaultStrT, DefaultIntT]): ...

   (
       NoNonDefaults ==
       NoNonDefaults[str] ==
       NoNonDefaults[str, int]
   )  # すべて有効


   class OneDefault(Generic[T, DefaultBoolT]): ...

   OneDefault[float] == OneDefault[float, bool]  # 有効
   reveal_type(OneDefault)          # 型は type[OneDefault[T, DefaultBoolT = bool]]
   reveal_type(OneDefault[float]()) # 型は OneDefault[float, bool]


   class AllTheDefaults(Generic[T1, T2, DefaultStrT, DefaultIntT, DefaultBoolT]): ...

   reveal_type(AllTheDefaults)                  # 型は type[AllTheDefaults[T1, T2, DefaultStrT = str, DefaultIntT = int, DefaultBoolT = bool]]
   reveal_type(AllTheDefaults[int, complex]())  # 型は AllTheDefaults[int, complex, str, int, bool]
   AllTheDefaults[int]  # 無効: AllTheDefaults に 2 つの引数が必要です
   (
       AllTheDefaults[int, complex] ==
       AllTheDefaults[int, complex, str] ==
       AllTheDefaults[int, complex, str, int] ==
       AllTheDefaults[int, complex, str, int, bool]
   )  # すべて有効

Python 3.12 のジェネリックの新しい構文 ( :pep:`695` によって導入) では、これをコンパイル時に強制できます::

   type Alias[DefaultT = int, T] = tuple[DefaultT, T]  # SyntaxError: デフォルトを持つ型変数の後に非デフォルトの型変数が続くことはできません

   def generic_func[DefaultT = int, T](x: DefaultT, y: T) -> None: ...  # SyntaxError: デフォルトを持つ型変数の後に非デフォルトの型変数が続くことはできません

   class GenericClass[DefaultT = int, T]: ...  # SyntaxError: デフォルトを持つ型変数の後に非デフォルトの型変数が続くことはできません

``ParamSpec`` のデフォルト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``ParamSpec`` のデフォルトは ``TypeVar``\ s と同じ構文を使用して定義されますが、``list`` 型のリストまたは省略記号リテラル「``...``」または別のスコープ内の ``ParamSpec`` を使用します ( `スコープルール`_ を参照)。

::

   DefaultP = ParamSpec("DefaultP", default=[str, int])

   class Foo(Generic[DefaultP]): ...

   reveal_type(Foo)                  # 型は type[Foo[DefaultP = [str, int]]]
   reveal_type(Foo())                # 型は Foo[[str, int]]
   reveal_type(Foo[[bool, bool]]())  # 型は Foo[[bool, bool]]

``TypeVarTuple`` のデフォルト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``TypeVarTuple`` のデフォルトは ``TypeVar``\ s と同じ構文を使用して定義されますが、単一の型の代わりにアンパックされたタプル型を使用します。または別のスコープ内の ``TypeVarTuple`` を使用します ( `スコープルール`_ を参照)。

::

   DefaultTs = TypeVarTuple("DefaultTs", default=Unpack[tuple[str, int]])

   class Foo(Generic[*DefaultTs]): ...

   reveal_type(Foo)               # 型は type[Foo[DefaultTs = *tuple[str, int]]]
   reveal_type(Foo())             # 型は Foo[str, int]
   reveal_type(Foo[int, bool]())  # 型は Foo[int, bool]

別の型パラメータを ``default`` として使用する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

これにより、ジェネリックの型パラメータが欠落しているが別の型パラメータが指定されている場合に、その値を再度使用できます。

別の型パラメータをデフォルトとして使用するには、``default`` と型パラメータが同じ型である必要があります (``TypeVar`` のデフォルトは ``TypeVar`` でなければなりません)。

::

   StartT = TypeVar("StartT", default=int)
   StopT = TypeVar("StopT", default=StartT)
   StepT = TypeVar("StepT", default=int | None)

   class slice(Generic[StartT, StopT, StepT]): ...

   reveal_type(slice)  # 型は type[slice[StartT = int, StopT = StartT, StepT = int | None]]
   reveal_type(slice())                        # 型は slice[int, int, int | None]
   reveal_type(slice[str]())                   # 型は slice[str, str, int | None]
   reveal_type(slice[str, bool, timedelta]())  # 型は slice[str, bool, timedelta]

   T2 = TypeVar("T2", default=DefaultStrT)

   class Foo(Generic[DefaultStrT, T2]):
       def __init__(self, a: DefaultStrT, b: T2) -> None: ...

   reveal_type(Foo(1, ""))  # 型は Foo[int, str]
   Foo[int](1, "")          # 無効: Foo[int, str] は Foo[int, int] の self に割り当てることはできません
   Foo[int]("", 1)          # 無効: Foo[str, int] は Foo[int, int] の self に割り当てることはできません

別の型パラメータをデフォルトとして使用する場合、次のルールが適用されます。 ``T1`` が ``T2`` のデフォルトであるとします。

スコープルール
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``T1`` はジェネリックのパラメータリストで ``T2`` の前に使用されなければなりません。

::

   T2 = TypeVar("T2", default=T1)

   class Foo(Generic[T1, T2]): ...   # 有効

   StartT = TypeVar("StartT", default="StopT")  # 前の例からデフォルトを入れ替えました
   StopT = TypeVar("StopT", default=int)
   class slice(Generic[StartT, StopT, StepT]): ...
                     # ^^^^^^ 無効: 順序が StopT をバインドすることを許可しません

外部スコープの型パラメータをデフォルトとして使用することはサポートされていません。

::

   class Foo(Generic[T1]):
       class Bar(Generic[T2]): ...   # 型エラー

バウンドルール
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``T1`` のバウンドは ``T2`` のバウンドに :term:`assignable` でなければなりません。

::

   T1 = TypeVar("T1", bound=int)
   TypeVar("Ok", default=T1, bound=float)     # 有効
   TypeVar("AlsoOk", default=T1, bound=int)   # 有効
   TypeVar("Invalid", default=T1, bound=str)  # 無効: int は str のサブタイプではありません

制約ルール
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``T2`` の制約は ``T1`` の制約のスーパーセットでなければなりません。

::

   T1 = TypeVar("T1", bound=int)
   TypeVar("Invalid", float, str, default=T1)         # 無効: 上限 int は制約 float または str と互換性がありません

   T1 = TypeVar("T1", int, str)
   TypeVar("AlsoOk", int, str, bool, default=T1)      # 有効
   TypeVar("AlsoInvalid", bool, complex, default=T1)  # 無効: {bool, complex} は {int, str} のスーパーセットではありません


ジェネリックのパラメータとしての型パラメータ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

型パラメータは、最初のパラメータがスコープ内であると判断される場合、``default`` 内のジェネリックのパラメータとして有効です ( `前のセクション <スコープルール_>`_ を参照)。

::

   T = TypeVar("T")
   ListDefaultT = TypeVar("ListDefaultT", default=list[T])

   class Bar(Generic[T, ListDefaultT]):
       def __init__(self, x: T, y: ListDefaultT): ...

   reveal_type(Bar)                         # 型は type[Bar[T, ListDefaultT = list[T]]]
   reveal_type(Bar[int])                    # 型は type[Bar[int, list[int]]]
   reveal_type(Bar[int](0, []))             # 型は Bar[int, list[int]]
   reveal_type(Bar[int, list[str]](0, []))  # 型は Bar[int, list[str]]
   reveal_type(Bar[int, str](0, ""))        # 型は Bar[int, str]

特殊化ルール
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ジェネリック型エイリアス
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

ジェネリック型エイリアスは、通常のサブスクリプションルールに従ってさらにサブスクリプションできます。 型パラメータにデフォルトがオーバーライドされていない場合、型パラメータが型エイリアスに置換されたかのように扱われるべきです。

::

   class SomethingWithNoDefaults(Generic[T, T2]): ...

   MyAlias: TypeAlias = SomethingWithNoDefaults[int, DefaultStrT]  # 有効
   reveal_type(MyAlias)          # 型は type[SomethingWithNoDefaults[int, DefaultStrT]]
   reveal_type(MyAlias[bool]())  # 型は SomethingWithNoDefaults[int, bool]

   MyAlias[bool, int]  # 無効: MyAlias に引数が多すぎます

サブクラス化
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型パラメータにデフォルトを持つジェネリッククラスは、ジェネリック型エイリアスと同様に動作します。 つまり、サブクラスは通常のサブスクリプションルールに従ってさらにサブスクリプションでき、オーバーライドされていないデフォルトは置換されるべきです。

::

   class SubclassMe(Generic[T, DefaultStrT]):
       x: DefaultStrT

   class Bar(SubclassMe[int, DefaultStrT]): ...
   reveal_type(Bar)          # 型は type[Bar[DefaultStrT = str]]
   reveal_type(Bar())        # 型は Bar[str]
   reveal_type(Bar[bool]())  # 型は Bar[bool]

   class Foo(SubclassMe[float]): ...

   reveal_type(Foo().x)  # 型は str

   Foo[str]  # 無効: Foo はさらにサブスクリプションできません

   class Baz(Generic[DefaultIntT, DefaultStrT]): ...

   class Spam(Baz): ...
   reveal_type(Spam())  # 型は <subclass of Baz[int, str]>

``bound`` と ``default`` の使用
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

``bound`` と ``default`` の両方が渡される場合、``default`` は ``bound`` に :term:`assignable` でなければなりません。 そうでない場合、型チェッカーはエラーを生成する必要があります。

::

   TypeVar("Ok", bound=float, default=int)     # 有効
   TypeVar("Invalid", bound=str, default=int)  # 無効: 上限とデフォルトが互換性がありません

制約
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

制約された ``TypeVar``\ s の場合、デフォルトは制約の 1 つである必要があります。 型チェッカーは、制約のサブタイプであってもエラーを生成する必要があります。

::

   TypeVar("Ok", float, str, default=float)     # 有効
   TypeVar("Invalid", float, str, default=int)  # 無効: float または str のいずれかが期待されましたが、int が得られました

関数のデフォルト
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

ジェネリック関数では、型チェッカーは型パラメータが何にも解決できない場合に型パラメータのデフォルトを使用することがあります。 この使用法のセマンティクスは指定されていません。型パラメータが解決できないすべてのコードパスで ``default`` が返されることを保証するのは実装が難しいためです。 型チェッカーはこのケースを許可しないか、サポートの実装を試みることができます。

::

   T = TypeVar('T', default=int)
   def func(x: int | set[T]) -> T: ...
   reveal_type(func(0))  # 型チェッカーはここで T のデフォルトの int を明らかにすることがあります

``TypeVarTuple`` の後に続くデフォルト
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型変数タプルの直後に続く型変数はデフォルトを持つことはできません。なぜなら、型引数が型変数タプルにバインドされるべきかデフォルトを持つ型変数にバインドされるべきかが曖昧になるためです。

::

   Ts = TypeVarTuple("Ts")
   T = TypeVar("T", default=bool)

   class Foo(Generic[*Ts, T]): ...  # 型チェッカーエラー

   # Ts = (int, str, float)、T = bool と解釈される可能性があります
   # または Ts = (int, str)、T = float と解釈される可能性があります
   Foo[int, str, float]

型パラメータのデフォルトを持つ ``TypeVarTuple`` の後に続く ``ParamSpec`` を持つことは許可されています。なぜなら、型引数が ``ParamSpec`` のものであるか型変数タプルのものであるかが曖昧になることはないためです。

::

   Ts = TypeVarTuple("Ts")
   P = ParamSpec("P", default=[float, bool])

   class Foo(Generic[*Ts, P]): ...  # 有効

   Foo[int, str]  # Ts = (int, str)、P = [float, bool]
   Foo[int, str, [bytes]]  # Ts = (int, str)、P = [bytes]

バインドルール
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

型パラメータのデフォルトは属性アクセス (呼び出しおよびサブスクリプションを含む) によってバインドされるべきです。

::

   class Foo[T = int]:
       def meth(self) -> Self:
           return self

   reveal_type(Foo.meth)  # 型は (self: Foo[int]) -> Foo[int]


.. _`self`:

``Self``
------------------------------------------------------------------------------------------

(元々 :pep:`673` によって指定されました。)

メソッドシグネチャでの使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

メソッドのシグネチャで使用される ``Self`` は、クラスにバインドされた ``TypeVar`` として扱われます。

::

    from typing import Self

    class Shape:
        def set_scale(self, scale: float) -> Self:
            self.scale = scale
            return self

これは次のものと同等に扱われます:

::

    from typing import TypeVar

    SelfShape = TypeVar("SelfShape", bound="Shape")

    class Shape:
        def set_scale(self: SelfShape, scale: float) -> SelfShape:
            self.scale = scale
            return self

これはサブクラスでも同様に機能します:

::

    class Circle(Shape):
        def set_radius(self, radius: float) -> Self:
            self.radius = radius
            return self

これは次のものと同等に扱われます:

::

    SelfCircle = TypeVar("SelfCircle", bound="Circle")

    class Circle(Shape):
        def set_radius(self: SelfCircle, radius: float) -> SelfCircle:
            self.radius = radius
            return self

1 つの実装戦略は、前処理ステップで前者を後者に単純にデシュガーすることです。 メソッドのシグネチャで ``Self`` が使用されている場合、メソッド内の ``self`` の型は ``Self`` になります。 他の場合、``self`` の型はエンクロージングクラスのままです。


クラスメソッドシグネチャでの使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``Self`` 型アノテーションは、操作するクラスのインスタンスを返すクラスメソッドにも役立ちます。 例えば、次のスニペットの ``from_config`` は、指定された ``config`` から ``Shape`` オブジェクトを構築します。

::

    class Shape:
        def __init__(self, scale: float) -> None: ...

        @classmethod
        def from_config(cls, config: dict[str, float]) -> Shape:
            return cls(config["scale"])


ただし、これにより ``Circle.from_config(...)`` は ``Shape`` 型の値を返すと推論されますが、実際には ``Circle`` であるべきです:

::

    class Circle(Shape):
        def circumference(self) -> float: ...

    shape = Shape.from_config({"scale": 7.0})
    # => Shape

    circle = Circle.from_config({"scale": 7.0})
    # => *Shape*、Circle ではありません

    circle.circumference()
    # エラー: `Shape` には `circumference` 属性がありません


現在の回避策は直感的ではなく、エラーが発生しやすいです:

::

    Self = TypeVar("Self", bound="Shape")

    class Shape:
        @classmethod
        def from_config(
            cls: type[Self], config: dict[str, float]
        ) -> Self:
            return cls(config["scale"])

代わりに、``Self`` を直接使用できます:

::

    from typing import Self

    class Shape:
        @classmethod
        def from_config(cls, config: dict[str, float]) -> Self:
            return cls(config["scale"])

これにより、複雑な ``cls: type[Self]`` アノテーションと ``bound`` を持つ ``TypeVar`` 宣言を回避できます。 再び、後者のコードは前者のコードと同等に動作します。

パラメータ型での使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``Self`` のもう 1 つの使用法は、現在のクラスのインスタンスを期待するパラメータをアノテートすることです:

::

    Self = TypeVar("Self", bound="Shape")

    class Shape:
        def difference(self: Self, other: Self) -> float: ...

        def apply(self: Self, f: Callable[[Self], None]) -> None: ...

``Self`` を直接使用して同じ動作を実現できます:

::

    from typing import Self

    class Shape:
        def difference(self, other: Self) -> float: ...

        def apply(self, f: Callable[[Self], None]) -> None: ...

``self: Self`` を指定することは無害であるため、次のように書く方が読みやすいと感じるユーザーもいるかもしれません:

::

    class Shape:
        def difference(self: Self, other: Self) -> float: ...

属性アノテーションでの使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``Self`` のもう 1 つの使用法は、属性をアノテートすることです。 1 つの例は、要素が現在のクラスに :term:`assignable` でなければならない ``LinkedList`` です。

::

    from dataclasses import dataclass
    from typing import Generic, TypeVar

    T = TypeVar("T")

    @dataclass
    class LinkedList(Generic[T]):
        value: T
        next: LinkedList[T] | None = None

    # OK
    LinkedList[int](value=1, next=LinkedList[int](value=2))
    # OK ではありません
    LinkedList[int](value=1, next=LinkedList[str](value="hello"))


ただし、``next`` 属性を ``LinkedList[T]`` とアノテートすると、サブクラスで無効な構築が許可されます:

::

    @dataclass
    class OrdinalLinkedList(LinkedList[int]):
        def ordinal_value(self) -> str:
            return as_ordinal(self.value)

    # LinkedList[int] は OrdinalLinkedList に割り当て可能ではないため、OK ではないはずですが、型チェッカーはこれを許可します。
    xs = OrdinalLinkedList(value=1, next=LinkedList[int](value=2))

    if xs.next:
        print(xs.next.ordinal_value())  # ランタイムエラー。


この制約は ``next: Self | None`` を使用して表現できます:

::

    from typing import Self

    @dataclass
    class LinkedList(Generic[T]):
        value: T
        next: Self | None = None

    @dataclass
    class OrdinalLinkedList(LinkedList[int]):
        def ordinal_value(self) -> str:
            return as_ordinal(self.value)

    xs = OrdinalLinkedList(value=1, next=LinkedList[int](value=2))
    # 型エラー: OrdinalLinkedList が期待されましたが、LinkedList[int] が得られました。

    if xs.next is not None:
        xs.next = OrdinalLinkedList(value=3, next=None)  # OK
        xs.next = LinkedList[int](value=3, next=None)  # OK ではありません



上記のコードは、各 ``Self`` 型を含む属性を ``Self | None`` を返す ``property`` として扱うことと同等です:

::

    from dataclasses import dataclass
    from typing import Any, Generic, TypeVar

    T = TypeVar("T")
    Self = TypeVar("Self", bound="LinkedList")


    class LinkedList(Generic[T]):
        value: T

        @property
        def next(self: Self) -> Self | None:
            return self._next

        @next.setter
        def next(self: Self, next: Self | None) -> None:
            self._next = next

    class OrdinalLinkedList(LinkedList[int]):
        def ordinal_value(self) -> str:
            return str(self.value)

ジェネリッククラスでの使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``Self`` はジェネリッククラスのメソッドでも使用できます:

::

    class Container(Generic[T]):
        value: T
        def set_value(self, value: T) -> Self: ...


これは次のものと同等です:

::

    Self = TypeVar("Self", bound="Container[Any]")

    class Container(Generic[T]):
        value: T
        def set_value(self: Self, value: T) -> Self: ...


この動作は、メソッドが呼び出されたオブジェクトの型引数を保持します。 具体的な型 ``Container[int]`` を持つオブジェクトで呼び出された場合、``Self`` は ``Container[int]`` にバインドされます。 ジェネリック型 ``Container[T]`` を持つオブジェクトで呼び出された場合、``Self`` は ``Container[T]`` にバインドされます:

::

    def object_with_concrete_type() -> None:
        int_container: Container[int]
        str_container: Container[str]
        reveal_type(int_container.set_value(42))  # => Container[int]
        reveal_type(str_container.set_value("hello"))  # => Container[str]

    def object_with_generic_type(
        container: Container[T], value: T,
    ) -> Container[T]:
        return container.set_value(value)  # => Container[T]


PEP は、メソッド ``set_value`` 内の ``self.value`` の正確な型を指定していません。 一部の型チェッカーは、クラスローカル型変数を使用して ``Self`` 型を実装し、``Self = TypeVar(“Self”, bound=Container[T])`` として推論することを選択できます。 ただし、クラスローカル型変数は標準化された型システム機能ではないため、``self.value`` に対して ``Any`` を推論することも許容されます。 これは型チェッカーに任せます。

``Self`` を型引数と一緒に使用することは拒否されることに注意してください。 これは、``self`` パラメータの型についての曖昧さを生み出し、不要な複雑さを導入するためです:

::

    class Container(Generic[T]):
        def foo(
            self, other: Self[int], other2: Self,
        ) -> Self[str]:  # 拒否されます
            ...

そのような場合、``self`` の型を明示的に使用することをお勧めします:

::

    class Container(Generic[T]):
        def foo(
            self: Container[T],
            other: Container[int],
            other2: Container[T]
        ) -> Container[str]: ...


プロトコルでの使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``Self`` はプロトコル内でも有効であり、クラス内での使用と同様です:

::

    from typing import Protocol, Self

    class ShapeProtocol(Protocol):
        scale: float

        def set_scale(self, scale: float) -> Self:
            self.scale = scale
            return self

これは次のものと同等に扱われます:

::

    from typing import TypeVar

    SelfShape = TypeVar("SelfShape", bound="ShapeProtocol")

    class ShapeProtocol(Protocol):
        scale: float

        def set_scale(self: SelfShape, scale: float) -> SelfShape:
            self.scale = scale
            return self


TypeVars がプロトコルにバインドされる場合の動作の詳細については、 :pep:`PEP 544 <544#self-types-in-protocols>` を参照してください。

プロトコルに対するクラスの割り当て可能性をチェックする: プロトコルがメソッドや属性アノテーションで ``Self`` を使用する場合、対応するメソッドや属性アノテーションが ``Self`` または ``Foo`` または ``Foo`` のサブクラスのいずれかを使用する場合、クラス ``Foo`` はプロトコルに :term:`assignable` です。 以下の例を参照してください:

::

    from typing import Protocol

    class ShapeProtocol(Protocol):
        def set_scale(self, scale: float) -> Self: ...

    class ReturnSelf:
        scale: float = 1.0

        def set_scale(self, scale: float) -> Self:
            self.scale = scale
            return self

    class ReturnConcreteShape:
        scale: float = 1.0

        def set_scale(self, scale: float) -> ReturnConcreteShape:
            self.scale = scale
            return self

    class BadReturnType:
        scale: float = 1.0

        def set_scale(self, scale: float) -> int:
            self.scale = scale
            return 42

    class ReturnDifferentClass:
        scale: float = 1.0

        def set_scale(self, scale: float) -> ReturnConcreteShape:
            return ReturnConcreteShape(...)

    def accepts_shape(shape: ShapeProtocol) -> None:
        y = shape.set_scale(0.5)
        reveal_type(y)

    def main() -> None:
        return_self_shape: ReturnSelf
        return_concrete_shape: ReturnConcreteShape
        bad_return_type: BadReturnType
        return_different_class: ReturnDifferentClass

        accepts_shape(return_self_shape)  # OK
        accepts_shape(return_concrete_shape)  # OK
        accepts_shape(bad_return_type)  # OK ではありません
        # 非サブクラスを返すため、OK ではありません。
        accepts_shape(return_different_class)


``Self`` の有効な場所
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``Self`` アノテーションはクラスコンテキストでのみ有効であり、常にエンクロージングクラスを参照します。 ネストされたクラスを含むコンテキストでは、``Self`` は常に最も内側のクラスを参照します。

次の ``Self`` の使用は受け入れられます:

::

    class ReturnsSelf:
        def foo(self) -> Self: ... # 受け入れられます

        @classmethod
        def bar(cls) -> Self:  # 受け入れられます
            return cls()

        def __new__(cls, value: int) -> Self: ...  # 受け入れられます

        def explicitly_use_self(self: Self) -> Self: ...  # 受け入れられます

        # 受け入れられます (Self は他の型内にネストできます)
        def returns_list(self) -> list[Self]: ...

        # 受け入れられます (Self は他の型内にネストできます)
        @classmethod
        def return_cls(cls) -> type[Self]:
            return cls

    class Child(ReturnsSelf):
        # 受け入れられます (Self アノテーションを使用するメソッドをオーバーライドできます)
        def foo(self) -> Self: ...

    class TakesSelf:
        def foo(self, other: Self) -> bool: ...  # 受け入れられます

    class Recursive:
        # 受け入れられます (Self | None を返す @property として扱われます)
        next: Self | None

    class CallableAttribute:
        def foo(self) -> int: ...

        # 受け入れられます (呼び出し可能な型を返す @property として扱われます)
        bar: Callable[[Self], int] = foo

    class HasNestedFunction:
        x: int = 42

        def foo(self) -> None:

            # 受け入れられます (Self は HasNestedFunction にバインドされます)。
            def nested(z: int, inner_self: Self) -> Self:
                print(z)
                print(inner_self.x)
                return inner_self

            nested(42, self)  # OK


    class Outer:
        class Inner:
            def foo(self) -> Self: ...  # 受け入れられます (Self は Inner にバインドされます)


次の ``Self`` の使用は拒否されます。

::

    def foo(bar: Self) -> Self: ...  # 拒否されます (クラス内ではありません)

    bar: Self  # 拒否されます (クラス内ではありません)

    class Foo:
        # 拒否されます (Self は不明と見なされます)。
        def has_existing_self_annotation(self: T) -> Self: ...

    class Foo:
        def return_concrete_type(self) -> Self:
            return Foo()  # 拒否されます (以下の FooChild を参照してください)

    class FooChild(Foo):
        child_value: int = 42

        def child_method(self) -> None:
            # 実行時には、これは Foo であり、FooChild ではありません。
            y = self.return_concrete_type()

            y.child_value
            # ランタイムエラー: Foo には child_value 属性がありません

    class Bar(Generic[T]):
        def bar(self) -> T: ...

    class Baz(Bar[Self]): ...  # 拒否されます

型チェッカーに多くの特別な処理を必要とするため、クラス定義の外部で ``Self`` を使用することは拒否されます。 クラス定義の外部で ``Self`` を使用することは PEP の他の部分に反するため、エイリアスの追加の利便性は追加の複雑さに見合わないと考えています:

::

    TupleSelf = Tuple[Self, Self]  # 拒否されます

    class Alias:
        def return_tuple(self) -> TupleSelf:  # 拒否されます
            return (self, self)

静的メソッドで ``Self`` を使用することは拒否されます。 ``Self`` は ``self`` や ``cls`` を返すことがないため、あまり価値がありません。 唯一の使用例は、パラメータ自体を返すか、パラメータとして渡されたコンテナから要素を返すことです。 これらは追加の複雑さに見合わないと考えています。

::

    class Base:
        @staticmethod
        def make() -> Self:  # 拒否されます
            ...

        @staticmethod
        def return_parameter(foo: Self) -> Self:  # 拒否されます
            ...

同様に、メタクラスで ``Self`` を使用することは拒否されます。 ``Self`` は常に同じ型 (``self`` の型) を参照します。 ただし、メタクラスでは、異なるメソッドシグネチャで異なる型を参照する必要があります。 例えば、``__mul__`` では、戻り値の型の ``Self`` はエンクロージングクラス ``MyMetaclass`` ではなく、実装クラス ``Foo`` を参照する必要があります。 ただし、``__new__`` では、戻り値の型の ``Self`` はエンクロージングクラス ``MyMetaclass`` を参照する必要があります。 混乱を避けるため、このエッジケースは拒否されます。

::

    class MyMetaclass(type):
        def __new__(cls, *args: Any) -> Self:  # 拒否されます
            return super().__new__(cls, *args)

        def __mul__(cls, count: int) -> list[Self]:  # 拒否されます
            return [cls()] * count

    class Foo(metaclass=MyMetaclass): ...

.. _`variance-inference`:

分散推論
------------------------------------------------------------------------------------------

(元々 :pep:`695` によって指定されました。)

Python 3.12 でジェネリッククラスの明示的な構文が導入されたことで、型パラメータの分散を指定する必要がなくなりました。 代わりに、型チェッカーはクラス内での使用に基づいて型パラメータの分散を推論します。 型パラメータは、その使用方法に応じて不変、共変、反変と推論されます。

Python 型チェッカーにはすでに、ジェネリックプロトコルクラス内の分散を検証する目的で型パラメータの分散を決定する機能が含まれています。 この機能は、プロトコルであるかどうかに関係なく、すべてのクラスに使用して各型パラメータの分散を計算できます。

型パラメータの分散を計算するアルゴリズムは次のとおりです。

ジェネリッククラスの各型パラメータについて:

1. 型パラメータが可変長 (``TypeVarTuple``) またはパラメータ仕様 (``ParamSpec``) の場合、それは常に不変と見なされます。 これ以上の推論は必要ありません。

2. 型パラメータが従来の ``TypeVar`` 宣言から派生し、``infer_variance`` として指定されていない場合、その分散は ``TypeVar`` コンストラクタ呼び出しによって指定されます。 これ以上の推論は必要ありません。

3. クラスの 2 つの特殊化バージョンを作成します。 これらを ``upper`` と ``lower`` の特殊化と呼びます。 これらの特殊化の両方で、推論される型パラメータ以外のすべての型パラメータをダミー型インスタンス (型パラメータの上限または制約を満たすと仮定される具体的な匿名クラス) に置き換えます。 ``upper`` 特殊化クラスでは、対象の型パラメータを ``object`` インスタンスで特殊化します。 この特殊化は型パラメータの上限や制約を無視します。 ``lower`` 特殊化クラスでは、対象の型パラメータをそのまま特殊化します (対応する型引数は型パラメータ自体です)。

4. 通常の割り当て可能性ルールを使用して ``lower`` が ``upper`` に割り当て可能かどうかを判断します。 そうである場合、対象の型パラメータは共変です。 そうでない場合、``upper`` が ``lower`` に割り当て可能かどうかを判断します。 そうである場合、対象の型パラメータは反変です。 これらの組み合わせのいずれも割り当て可能でない場合、対象の型パラメータは不変です。

例を示します。

::

    class ClassA[T1, T2, T3](list[T1]):
        def method1(self, a: T2) -> None:
            ...

        def method2(self) -> T3:
            ...

``T1`` の分散を判断するために、次のように ``ClassA`` を特殊化します:

::

    upper = ClassA[object, Dummy, Dummy]
    lower = ClassA[T1, Dummy, Dummy]

``upper`` は ``lower`` に割り当て可能ではないことがわかります。 同様に、``lower`` は ``upper`` に割り当て可能ではないため、``T1`` は不変と結論付けます。

``T2`` の分散を判断するために、次のように ``ClassA`` を特殊化します:

::

    upper = ClassA[Dummy, object, Dummy]
    lower = ClassA[Dummy, T2, Dummy]

``upper`` は ``lower`` に割り当て可能であるため、``T2`` は反変です。

``T3`` の分散を判断するために、次のように ``ClassA`` を特殊化します:

::

    upper = ClassA[Dummy, Dummy, object]
    lower = ClassA[Dummy, Dummy, T3]

``lower`` は ``upper`` に割り当て可能であるため、``T3`` は共変です。


TypeVar の自動分散
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

既存の ``TypeVar`` クラスコンストラクタは、``covariant`` および ``contravariant`` という名前のキーワードパラメータを受け入れます。 これらの両方が ``False`` の場合、型変数は不変と見なされます。 PEP 695 は、型チェッカーが型変数が不変、共変、反変であるかどうかを推論することを示す ``infer_variance`` という名前のキーワードパラメータを追加します。 対応するインスタンス変数 ``__infer_variance__`` は、ランタイムでアクセスして分散が推論されるかどうかを判断できます。 新しい構文を使用して暗黙的に割り当てられる型変数は常に ``__infer_variance__`` が ``True`` に設定されます。

従来の構文を使用するジェネリッククラスは、明示的な分散と推論された分散を持つ型変数の組み合わせを含むことができます。

::

    T1 = TypeVar("T1", infer_variance=True)  # 推論された分散
    T2 = TypeVar("T2")  # 不変
    T3 = TypeVar("T3", covariant=True)  # 共変

    # 型チェッカーは T1 の分散を推論する必要がありますが、T2 および T3 の指定された分散を使用する必要があります。
    class ClassA(Generic[T1, T2, T3]): ...


従来の TypeVars との互換性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``TypeVar``、``TypeVarTuple``、および ``ParamSpec`` を割り当てる既存のメカニズムは、後方互換性のために保持されます。 ただし、これらの「従来の」型変数は、新しい構文を使用して割り当てられた型パラメータと組み合わせるべきではありません。 このような組み合わせは型チェッカーによってエラーとしてフラグを立てる必要があります。 これは、型パラメータの順序が曖昧であるためです。

クラス、関数、または型エイリアスが新しい構文を使用しない場合、従来の型変数を新しいスタイルの型パラメータと組み合わせることは問題ありません。 新しいスタイルの型パラメータは外部スコープから派生する必要があります。

::

    K = TypeVar("K")

    class ClassA[V](dict[K, V]): ...  # 型チェッカーエラー

    class ClassB[K, V](dict[K, V]): ...  # OK

    class ClassC[V]:
        # "method1" の K および V の使用は問題ありません。これは "従来の" ジェネリック関数メカニズムを使用しており、型パラメータが暗黙的であるためです。この場合、V は外部スコープ (ClassC) から派生し、K は "method1" の型パラメータとして暗黙的に導入されます。
        def method1(self, a: V, b: K) -> V | K: ...

        # "method2" の M および K の使用は許可されていません。このメソッドは型パラメータの新しい構文を使用しており、メソッドに関連付けられたすべての型パラメータは明示的に宣言されている必要があります。この場合、``K`` は "method2" によって宣言されておらず、外部スコープで定義された新しいスタイルの型パラメータによって提供されていません。
        def method2[M](self, a: M, b: K) -> M | K: ...
