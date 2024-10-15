.. _protocols:

プロトコル
------------------------------------------------------------------------------------------

(元々 :pep:`544` で指定されています。)

用語
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*プロトコル* という用語は、:term:`structural` サブタイピングをサポートするいくつかの型に使用されます。 その理由は、例えば *イテレータープロトコル* という用語がコミュニティで広く理解されており、この概念に対して静的に型付けされたコンテキストで新しい用語を作成すると混乱を招くだけだからです。

この欠点は、*プロトコル* という用語が 2 つの微妙に異なる意味で過負荷になることです。 1 つ目は、イテレーターなどのプロトコルの伝統的でよく知られたがやや曖昧な概念です。 2 つ目は、静的に型付けされたコードのプロトコルのより明確に定義された概念です。 この区別はほとんどの場合重要ではなく、他の場合には静的型の概念を指すときに *プロトコルクラス* などの修飾子を追加するだけです。

クラスがプロトコルをその MRO に含む場合、そのクラスはプロトコルの *明示的* サブクラスと呼ばれます。 クラスがプロトコルのすべての属性とメソッドを、そのプロトコルの属性とメソッドの型に :term:`assignable` な型で定義している場合、そのクラスはプロトコルを実装しており、プロトコルに代入可能であると言われます。 クラスがプロトコルに代入可能であるが、プロトコルが MRO に含まれていない場合、そのクラスはプロトコルに *暗黙的に* 代入可能です。 (プロトコル属性がサブクラスで ``None`` に設定されている場合、プロトコルを明示的にサブクラス化してもそれを実装しないことに注意してください。 詳細については、Python :py:ref:`データモデル <specialnames>` を参照してください。)

プロトコルに代入可能であるために他のクラスに必須のプロトコルの属性 (変数とメソッド) は「プロトコルメンバー」と呼ばれます。

.. _protocol-definition:

プロトコルの定義
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

プロトコルは、通常はリストの最後にある基本クラスリストに :term:`special form` ``typing.Protocol`` (``abc.ABCMeta`` のインスタンス) を含めることによって定義されます。 ここに簡単な例があります::

  from typing import Protocol

  class SupportsClose(Protocol):
      def close(self) -> None:
          ...

今、``SupportsClose.close`` に :term:`assignable` な型シグネチャを持つ ``close()`` メソッドを持つクラス ``Resource`` を定義すると、プロトコル型に対して :term:`structural` 代入可能性が使用されるため、暗黙的に ``SupportsClose`` に代入可能になります::

  class Resource:
      ...
      def close(self) -> None:
          self.file.close()
          self.lock.release()

以下に明示的に記載されているいくつかの制限を除いて、プロトコル型は通常の型が使用できるすべてのコンテキストで使用できます::

  def close_all(things: Iterable[SupportsClose]) -> None:
      for t in things:
          t.close()

  f = open('foo.txt')
  r = Resource()
  close_all([f, r])  # OK!
  close_all([1])     # エラー: 'int' には 'close' メソッドがありません

ユーザー定義のクラス ``Resource`` と組み込みの ``IO`` 型 (``open()`` の戻り値の型) の両方が ``SupportsClose`` に代入可能であることに注意してください。 これは、それぞれが代入可能な型シグネチャを持つ ``close()`` メソッドを提供するためです。


プロトコルメンバー
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

プロトコルクラス本体で定義されたすべてのメソッドは、通常のメソッドと ``@abstractmethod`` で装飾されたメソッドの両方がプロトコルメンバーです。 プロトコルメソッドのパラメータに注釈がない場合、それらの型は ``Any`` と見なされます (see :ref:`"The meaning of annotations" <missing-annotations>`)。 プロトコルメソッドの本体は型チェックされます。 ``super()`` 経由で呼び出されるべきではない抽象メソッドは ``NotImplementedError`` を発生させる必要があります。 例::

  from typing import Protocol
  from abc import abstractmethod

  class Example(Protocol):
      def first(self) -> int:     # これはプロトコルメンバーです
          return 42

      @abstractmethod
      def second(self) -> int:    # デフォルト実装のないメソッド
          raise NotImplementedError

静的メソッド、クラスメソッド、およびプロパティもプロトコルで許可されています。

プロトコル変数を定義するには、クラス本体で変数注釈を使用できます。 メソッドの本体で ``self`` を介して代入によってのみ定義された追加の属性は許可されません。 その理由は、プロトコルクラスの実装がサブタイプによって共有されることはほとんどないため、インターフェースはデフォルトの実装に依存すべきではないからです。 例::

  from typing import Protocol

  class Template(Protocol):
      name: str        # これはプロトコルメンバーです
      value: int = 0   # これも (デフォルト付き)

      def method(self) -> None:
          self.temp: list[int] = [] # 型チェッカーのエラー

  class Concrete:
      def __init__(self, name: str, value: int) -> None:
          self.name = name
          self.value = value

      def method(self) -> None:
          return

  var: Template = Concrete('value', 42)  # OK

プロトコルクラス変数とプロトコルインスタンス変数を区別するために、特別な :ref:`ClassVar <classvar>` 注釈を使用する必要があります。 上記で定義されたプロトコル変数はデフォルトで読み取りおよび書き込み可能と見なされます。 読み取り専用のプロトコル変数を定義するには、(抽象) プロパティを使用できます。


実装の明示的な宣言
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

特定のクラスが特定のプロトコルを実装していることを明示的に宣言するには、通常の基本クラスとして使用できます。 この場合、クラスはプロトコルメンバーのデフォルト実装を使用できます。 静的解析ツールは、クラスが特定のプロトコルを実装していることを自動的に検出することが期待されます。 したがって、プロトコルを明示的にサブクラス化することは可能ですが、型チェックのためにそれを行う必要は *ありません*。

代入可能な関係が暗黙的であり、:term:`structural` である場合、デフォルトの実装は使用できません。 継承のセマンティクスは変更されません。 例::

    class PColor(Protocol):
        @abstractmethod
        def draw(self) -> str:
            ...
        def complex_method(self) -> int:
            # ここに複雑なコード

    class NiceColor(PColor):
        def draw(self) -> str:
            return "deep blue"

    class BadColor(PColor):
        def draw(self) -> str:
            return super().draw()  # エラー、デフォルト実装がありません

    class ImplicitColor:   # ここに 'PColor' ベースがないことに注意してください
        def draw(self) -> str:
            return "probably gray"
        def complex_method(self) -> int:
            # クラスはこれを実装する必要があります

    nice: NiceColor
    another: ImplicitColor

    def represent(c: PColor) -> None:
        print(c.draw(), c.complex_method())

    represent(nice) # OK
    represent(another) # これも OK

明示的なサブクラス化とプロトコルの暗黙的な実装の違いはほとんどないことに注意してください。 明示的なサブクラス化の主な利点は、いくつかのプロトコルメソッドを「無料で」取得できることです。 さらに、型チェッカーはクラスが実際にプロトコルを正しく実装していることを静的に検証できます::

    class RGB(Protocol):
        rgb: tuple[int, int, int]

        @abstractmethod
        def intensity(self) -> int:
            return 0

    class Point(RGB):
        def __init__(self, red: int, green: int, blue: str) -> None:
            self.rgb = red, green, blue  # エラー、'blue' は 'int' でなければなりません

        # 型チェッカーは 'intensity' が定義されていないことを警告するかもしれません

クラスは複数のプロトコルと通常のクラスを明示的に継承できます。 この場合、メソッドは通常の MRO を使用して解決され、型チェッカーはすべてのメンバーの代入可能性が正しいことを検証します。 ``@abstractmethod`` のセマンティクスは変更されません。 明示的なサブクラスによってすべて実装される必要があります。


プロトコルのマージと拡張
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

一般的な哲学は、プロトコルは通常の ABC とほぼ同じですが、静的型チェッカーはそれらを特別に処理します。 プロトコルクラスをサブクラス化しても、``typing.Protocol`` が明示的な基本クラスとしても含まれていない限り、サブクラスはプロトコルにはなりません。 この基本がない場合、クラスは :term:`structural` サブタイピングで使用できない通常の ABC に「ダウングレード」されます。 このルールの根拠は、基本クラスの 1 つがプロトコルであるために、あるクラスがプロトコルとして機能することを偶然に持たせたくないからです。 静的型付けの世界では、依然として :term:`nominal` サブタイピングを構造的サブタイピングよりもわずかに好みます。

*プロトコル* を即時基本クラスとして持ち、即時基本クラスとして ``typing.Protocol`` も持つことによってサブプロトコルを定義できます::

  from typing import Protocol
  from collections.abc import Sized

  class SizedAndClosable(Sized, Protocol):
      def close(self) -> None:
          ...

今、プロトコル ``SizedAndClosable`` は 2 つのメソッド ``__len__`` と ``close`` を持つプロトコルです。 基本クラスリストに ``Protocol`` を省略すると、これは ``Sized`` を実装する必要がある通常の (非プロトコル) クラスになります。 あるいは、`protocol-definition`_ セクションの例から ``SupportsClose`` プロトコルを ``typing.Sized`` とマージすることによって ``SizedAndClosable`` プロトコルを実装できます::

  from collections.abc import Sized

  class SupportsClose(Protocol):
      def close(self) -> None:
          ...

  class SizedAndClosable(Sized, SupportsClose, Protocol):
      pass

``SizedAndClosable`` の 2 つの定義は同等です。 プロトコル間のサブクラス関係は、MRO ではなく :term:`structural` :term:`代入可能性 <assignable>` が基準であるため、代入可能性を考慮する場合には意味がありません。

基本クラスリストに ``Protocol`` が含まれている場合、他のすべての基本クラスはプロトコルでなければなりません。 プロトコルは通常のクラスを拡張できません。 明示的なサブクラス化に関するルールは、少なくとも 1 つの抽象メソッドが未実装であることによって抽象性が単に定義される通常の ABC とは異なることに注意してください。 プロトコルクラスは *明示的に* マークされなければなりません。


ジェネリックプロトコル
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ジェネリックプロトコルは重要です。 例えば、``SupportsAbs``、``Iterable``、および ``Iterator`` はジェネリックプロトコルです。 それらは通常の非プロトコルジェネリック型と同様に定義されます::

  class Iterable(Protocol[T]):
      @abstractmethod
      def __iter__(self) -> Iterator[T]:
          ...

``Protocol[T, S, ...]`` は ``Protocol, Generic[T, S, ...]`` の省略形として許可されます。

ユーザー定義のジェネリックプロトコルは明示的に宣言された分散をサポートします。 型チェッカーは、推論された分散が宣言された分散と異なる場合に警告します。 例::

  T = TypeVar('T')
  T_co = TypeVar('T_co', covariant=True)
  T_contra = TypeVar('T_contra', contravariant=True)

  class Box(Protocol[T_co]):
      def content(self) -> T_co:
          ...

  box: Box[float]
  second_box: Box[int]
  box = second_box  # これは 'Box' の共変性のために OK です。

  class Sender(Protocol[T_contra]):
      def send(self, data: T_contra) -> int:
          ...

  sender: Sender[float]
  new_sender: Sender[int]
  new_sender = sender  # OK、'Sender' は反変です。

  class Proto(Protocol[T]):
      attr: T  # このクラスは可変属性を持つため不変です

  var: Proto[float]
  another_var: Proto[int]
  var = another_var  # エラー! 'Proto[float]' は 'Proto[int]' に代入できません。

名義クラスとは異なり、事実上の共変プロトコルは不変として宣言できません。 これは、これがサブタイピングの推移性を破る可能性があるためです。 例えば::

  T = TypeVar('T')

  class AnotherBox(Protocol[T]):  # エラー、このプロトコルは T において共変であり、不変ではありません。
      def content(self) -> T:
          ...


再帰プロトコル
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

再帰プロトコルもサポートされています。 プロトコルクラス名への前方参照は :ref:`文字列として与えることができます <forward-references>`。 再帰プロトコルは、ツリーのような自己参照データ構造を抽象的に表現するのに役立ちます::

  class Traversable(Protocol):
      def leaves(self) -> Iterable['Traversable']:
          ...

再帰プロトコルの場合、決定が自分自身に依存する状況では、クラスはプロトコルに代入可能と見なされることに注意してください。 前の例を続けます::

  class SimpleTree:
      def leaves(self) -> list['SimpleTree']:
          ...

  root: Traversable = SimpleTree()  # OK

  class Tree(Generic[T]):
      def leaves(self) -> list['Tree[T]']:
          ...

  def walk(graph: Traversable) -> None:
      ...
  tree: Tree[float] = Tree()
  walk(tree)  # OK、'Tree[float]' は 'Traversable' に代入可能です


プロトコルにおける自己型
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

プロトコルにおける自己型は、:ref:`他のメソッドのルール <annotating-methods>` に従います。 例えば::

  C = TypeVar('C', bound='Copyable')
  class Copyable(Protocol):
      def copy(self: C) -> C:

  class One:
      def copy(self) -> 'One':
          ...

  T = TypeVar('T', bound='Other')
  class Other:
      def copy(self: T) -> T:
          ...

  c: Copyable
  c = One()  # OK
  c = Other()  # これも OK

他の型との代入可能性の関係
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

プロトコルはインスタンス化できないため、ランタイム型がプロトコルである値はありません。 プロトコル型の変数とパラメータについては、代入可能性の関係は次のルールに従います:

* プロトコルは具体的な型に代入可能ではありません。
* 具体的な型 ``X`` は、``X`` が ``P`` のすべてのプロトコルメンバーを代入可能な型で実装している場合にのみ、プロトコル ``P`` に代入可能です。 言い換えれば、プロトコルに関する :term:`代入可能性 <assignable>` は常に :term:`structural` です。
* プロトコル ``P1`` は、``P1`` が代入可能な型で ``P2`` のすべてのプロトコルメンバーを定義している場合にのみ、他のプロトコル ``P2`` に代入可能です。

ジェネリックプロトコル型は、非プロトコル型と同じ分散ルールに従います。 プロトコル型は、ユニオン、``ClassVar``、型変数の境界など、他の型が使用できるすべてのコンテキストで使用できます。 ジェネリックプロトコルは、継承関係によって定義された代入可能性の代わりに構造的代入可能性を使用することを除いて、ジェネリック抽象クラスのルールに従います。

静的型チェッカーは、対応するプロトコルが *インポートされていなくても* プロトコルの実装を認識します::

  # file lib.py
  from collections.abc import Sized

  T = TypeVar('T', contravariant=True)
  class ListLike(Sized, Protocol[T]):
      def append(self, x: T) -> None:
          pass

  def populate(lst: ListLike[int]) -> None:
      ...

  # file main.py
  from lib import populate  # ListLike がインポートされていないことに注意してください

  class MockStack:
      def __len__(self) -> int:
          return 42
      def append(self, x: int) -> None:
          print(x)

  populate([1, 2, 3])    # 型チェックを通過
  populate(MockStack())  # これも OK


プロトコルのユニオンとインターセクション
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

プロトコルクラスのユニオンは、非プロトコルクラスと同じ方法で動作します。 例えば::

  from typing import Protocol

  class Exitable(Protocol):
      def exit(self) -> int:
          ...
  class Quittable(Protocol):
      def quit(self) -> int | None:
          ...

  def finish(task: Exitable | Quittable) -> int:
      ...
  class DefaultJob:
      ...
      def quit(self) -> int:
          return 0
  finish(DefaultJob()) # OK

プロトコルのインターセクションを定義するには、多重継承を使用できます。 例::

  from collections.abc import Iterable, Hashable

  class HashableFloats(Iterable[float], Hashable, Protocol):
      pass

  def cached_func(args: HashableFloats) -> float:
      ...
  cached_func((1, 2, 3)) # OK、タプルはハッシュ可能であり、反復可能です


``type[]`` とクラスオブジェクト vs プロトコル
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``type[Proto]`` で注釈された変数とパラメータは、``Proto`` の具体的な (非プロトコル) :term:`一貫したサブタイプ <consistent subtype>` のみを受け入れます。 これの主な理由は、そのような型のパラメータのインスタンス化を許可することです。 例えば::

  class Proto(Protocol):
      @abstractmethod
      def meth(self) -> int:
          ...
  class Concrete:
      def meth(self) -> int:
          return 42

  def fun(cls: type[Proto]) -> int:
      return cls().meth() # OK
  fun(Proto)              # エラー
  fun(Concrete)           # OK

同じルールが変数にも適用されます::

  var: Type[Proto]
  var = Proto    # エラー
  var = Concrete # OK
  var().meth()   # OK

変数が明示的に型付けされていない場合、ABC またはプロトコルクラスを変数に代入することは許可されており、そのような代入は型エイリアスを作成します。 通常の (非抽象) クラスの場合、``type[]`` の動作は変更されません。

クラスオブジェクトは、すべてのメンバーにアクセスすると、プロトコルメンバーの型に代入可能な型が得られる場合、プロトコルの実装と見なされます。 例えば::

  from typing import Any, Protocol

  class ProtoA(Protocol):
      def meth(self, x: int) -> int: ...
  class ProtoB(Protocol):
      def meth(self, obj: Any, x: int) -> int: ...

  class C:
      def meth(self, x: int) -> int: ...

  a: ProtoA = C  # 型チェックエラー、シグネチャが一致しません!
  b: ProtoB = C  # OK


``NewType()`` と型エイリアス
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

プロトコルは本質的に匿名です。 この点を強調するために、静的型チェッカーは、特定の型が提供されるという幻想を避けるために、``NewType()`` 内のプロトコルクラスを拒否する場合があります::

  from typing import NewType, Protocol
  from collections.abc import Iterator

  class Id(Protocol):
      code: int
      secrets: Iterator[bytes]

  UserId = NewType('UserId', Id)  # エラー、特定の型を提供できません

対照的に、型エイリアスは完全にサポートされています。 ジェネリック型エイリアスも含まれます::

  from typing import TypeVar
  from collections.abc import Reversible, Iterable, Sized

  T = TypeVar('T')
  class SizedIterable(Iterable[T], Sized, Protocol):
      pass
  CompatReversible = Reversible[T] | SizedIterable[T]


プロトコルの実装としてのモジュール
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

モジュールオブジェクトは、期待されるプロトコルに代入可能な場合、プロトコルが期待される場所で受け入れられます。 例えば::

  # file default_config.py
  timeout = 100
  one_flag = True
  other_flag = False

  # file main.py
  import default_config
  from typing import Protocol

  class Options(Protocol):
      timeout: int
      one_flag: bool
      other_flag: bool

  def setup(options: Options) -> None:
      ...

  setup(default_config)  # OK

モジュールレベルの関数の代入可能性を判断するために、対応するプロトコルメソッドの ``self`` 引数は削除されます。 例えば::

  # callbacks.py
  def on_error(x: int) -> None:
      ...
  def on_success() -> None:
      ...

  # main.py
  import callbacks
  from typing import Protocol

  class Reporter(Protocol):
      def on_error(self, x: int) -> None:
          ...
      def on_success(self) -> None:
          ...

  rp: Reporter = callbacks  # 型チェックを通過

.. _`runtime-checkable`:

``@runtime_checkable`` デコレータと ``isinstance()`` による型の絞り込み
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

デフォルトのセマンティクスは、プロトコル型に対して ``isinstance()`` および ``issubclass()`` が失敗することです。 これはダックタイピングの精神に基づいています。 プロトコルは基本的にダックタイピングを静的にモデル化するために使用され、ランタイムで明示的に使用されるわけではありません。

ただし、これが意味をなす場合、プロトコル型がカスタムインスタンスおよびクラスチェックを実装できるはずです。 これは、``collections.abc`` および ``typing`` の ``Iterable`` などの ABC がすでに行っているのと同様です。 ただし、これは非ジェネリックおよび非サブスクリプトジェネリックプロトコル (``Iterable`` は静的には ``Iterable[Any]`` と同等) に限定されます。 ``typing`` モジュールは、クラスおよびインスタンスチェックに対して ``collections.abc`` クラスと同じセマンティクスを提供する特別な ``@runtime_checkable`` クラスデコレータを定義します。 これにより、それらは「ランタイムプロトコル」になります::

  from typing import runtime_checkable, Protocol

  @runtime_checkable
  class SupportsClose(Protocol):
      def close(self):
          ...

  assert isinstance(open('some/file'), SupportsClose)

インスタンスチェックは静的には 100% 信頼できないことに注意してください。 これがこの動作がオプトインである理由です。 型チェッカーができる最善のことは、``isinstance(obj, Iterator)`` を ``hasattr(x, '__iter__') and hasattr(x, '__next__')`` を書くための簡単な方法として扱うことです。 この機能のリスクを最小限に抑えるために、次のルールが適用されます。

**定義**:

* *データおよび非データプロトコル*: プロトコルがメンバーとしてメソッドのみを含む場合 (例えば ``Sized``、``Iterator`` など)、そのプロトコルは非データプロトコルと呼ばれます。 少なくとも 1 つの非メソッドメンバー (例えば ``x: int``) を含むプロトコルはデータプロトコルと呼ばれます。
* *安全でない重複*: 型 ``X`` がプロトコル ``P`` と安全でない重複を持つと呼ばれるのは、``X`` が ``P`` に代入可能ではないが、すべてのメンバーが ``Any`` 型を持つプロトコル ``P`` の型消去バージョンに代入可能である場合です。 さらに、少なくとも 1 つの要素がプロトコル ``P`` と安全でない重複を持つユニオンがある場合、そのユニオン全体がプロトコル ``P`` と安全でない重複を持つと見なされます。

**仕様**:

* プロトコルは、``@runtime_checkable`` デコレータによって明示的にオプトインされている場合にのみ、``isinstance()`` および ``issubclass()`` の 2 番目の引数として使用できます。 この要件は、プロトコルチェックが動的に設定された属性の場合に型安全ではないため、および型チェッカーが ``isinstance()`` チェックが特定のクラスに対して安全であることを証明できるのは、そのクラスのすべてのサブクラスに対してではないためです。
* ``isinstance()`` はデータおよび非データプロトコルの両方で使用できますが、``issubclass()`` は非データプロトコルでのみ使用できます。 この制限は、いくつかのデータ属性がコンストラクタでインスタンスに設定される可能性があり、この情報がクラスオブジェクトで常に利用できるわけではないためです。
* 型チェッカーは、最初の引数の型とプロトコルの間に安全でない重複がある場合、``isinstance()`` または ``issubclass()`` 呼び出しを拒否する必要があります。
* 型チェッカーは、安全な ``isinstance()`` または ``issubclass()`` 呼び出しの後にユニオンから正しい要素を選択できる必要があります。 非ユニオン型からの絞り込みについては、型チェッカーは最善の判断を使用できます (これは意図的に指定されていません。正確な仕様は交差型を必要とするためです)。
