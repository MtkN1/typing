.. _`type-aliases`:

型エイリアス
==========================================================================================

（「TypeAlias」の導入については :pep:`613` を参照し、「type」ステートメントについては :pep:`695` を参照してください。）

型エイリアスは、単純な変数の代入によって定義できます::

  Url = str

  def retry(url: Url, retry_count: int) -> None: ...

または ``typing.TypeAlias`` を使用して::

  from typing import TypeAlias

  Url: TypeAlias = str

  def retry(url: Url, retry_count: int) -> None: ...

または ``type`` ステートメントを使用して（Python 3.12 以降）::

  type Url = str

  def retry(url: Url, retry_count: int) -> None: ...

型エイリアス名はユーザー定義の型を表すため、通常は大文字で始めることをお勧めします。

型エイリアスは、注釈内の型ヒントと同じくらい複雑にすることができます。つまり、型エイリアスとして許容されるものは、注釈としても許容されます::

    from typing import TypeVar
    from collections.abc import Iterable

    T = TypeVar('T', bound=float)
    Vector = Iterable[tuple[T, T]]

    def inproduct(v: Vector[T]) -> T:
        return sum(x*y for x, y in v)
    def dilate(v: Vector[T], scale: T) -> Vector[T]:
        return ((x * scale, y * scale) for x, y in v)
    vec: Vector[float] = []

これは次のように等価です::

    from typing import TypeVar
    from collections.abc import Iterable

    T = TypeVar('T', bound=float)

    def inproduct(v: Iterable[tuple[T, T]]) -> T:
        return sum(x*y for x, y in v)
    def dilate(v: Iterable[tuple[T, T]], scale: T) -> Iterable[tuple[T, T]]:
        return ((x * scale, y * scale) for x, y in v)
    vec: Iterable[tuple[float, float]] = []

.. _`typealias`:

``TypeAlias``
------------------------------------------------------------------------------------------

明示的なエイリアス宣言の構文を使用すると、3 つの可能な代入の種類（型付きグローバル式、型なしグローバル式、および型エイリアス）を明確に区別できます。これにより、注釈が追加されたときに型チェックが壊れる代入の存在を回避し、代入の性質を値の型に基づいて分類することを回避できます。

暗黙の構文（既存の構文）::

  x = 1  # 型なしグローバル式
  x: int = 1  # 型付きグローバル式

  x = int  # 型エイリアス
  x: type[int] = int  # 型付きグローバル式

明示的な構文::

  x = 1  # 型なしグローバル式
  x: int = 1  # 型付きグローバル式

  x = int  # 型なしグローバル式（以下の注に注意）
  x: type[int] = int  # 型付きグローバル式

  x: TypeAlias = int  # 型エイリアス
  x: TypeAlias = "MyClass"  # 型エイリアス

注: 上記の例は、暗黙的および明示的なエイリアス宣言を個別に示しています。後方互換性のために、型チェッカーは両方を同時にサポートする必要があり、型なしグローバル式 ``x = int`` も有効な型エイリアスと見なされます。

.. _`type-statement`:

``type`` ステートメント
------------------------------------------------------------------------------------------

型エイリアスは ``type`` ステートメントを使用して定義することもできます（Python 3.12 以降）。

``type`` ステートメントを使用すると、明示的にジェネリックな型エイリアスを作成できます::

  type ListOrSet[T] = list[T] | set[T]

ジェネリック型エイリアスの一部として宣言された型パラメータは、型エイリアスの右辺を評価する場合にのみ有効です。

``typing.TypeAlias`` と同様に、型チェッカーは右辺の式を型注釈内で許可される式形式に制限する必要があります。より複雑な式形式（呼び出し式、三項演算子、算術演算子、比較演算子など）の使用はエラーとしてフラグを立てる必要があります。

型エイリアス式は、従来の型変数（つまり、明示的な ``TypeVar`` コンストラクタ呼び出しで割り当てられたもの）を使用することはできません。この場合、型チェッカーはエラーを生成する必要があります。

::

    T = TypeVar("T")
    type MyList = list[T]  # 型チェッカーエラー: 従来の型変数の使用

.. _`newtype`:

``NewType``
------------------------------------------------------------------------------------------

プログラマーが単純なクラスを作成して論理エラーを回避したい場合もあります。例えば::

  class UserId(int):
      pass

  def get_by_user_id(user_id: UserId):
      ...

ただし、このアプローチはランタイムオーバーヘッドを導入します。これを回避するために、``typing.py`` は ``NewType`` というヘルパー関数を提供し、ほぼゼロのランタイムオーバーヘッドで単純なユニークな型を作成します。静的型チェッカーにとって、``Derived = NewType('Derived', Base)`` は次の定義とほぼ同等です::

  class Derived(Base):
      def __init__(self, _x: Base) -> None:
          ...

一方、ランタイムでは、``NewType('Derived', Base)`` は単に引数を返すダミー関数を返します。型チェッカーは、``UserId`` が期待される場所で ``int`` からの明示的なキャストを要求し、``int`` が期待される場所で ``UserId`` からの暗黙的なキャストを許可します。例::

        UserId = NewType('UserId', int)

        def name_by_id(user_id: UserId) -> str:
            ...

        UserId('user')          # 型チェックエラー

        name_by_id(42)          # 型チェックエラー
        name_by_id(UserId(42))  # OK

        num = UserId(5) + 1     # 型: int

``NewType`` は正確に 2 つの引数を受け入れます: 新しいユニークな型の名前と基本クラス。後者は適切なクラス（つまり、``Union`` などの型構築子ではなく）である必要があり、または ``NewType`` を呼び出して作成された別のユニークな型である必要があります。``NewType`` によって返される関数は 1 つの引数のみを受け入れます。これは、基本クラスのインスタンスを受け入れる 1 つのコンストラクタのみをサポートすることと同等です（上記参照）。例::

  class PacketId:
      def __init__(self, major: int, minor: int) -> None:
          self._major = major
          self._minor = minor

  TcpPacketId = NewType('TcpPacketId', PacketId)

  packet = PacketId(100, 100)
  tcp_packet = TcpPacketId(packet)  # OK

  tcp_packet = TcpPacketId(127, 0)  # 型チェッカーエラーおよびランタイムエラー

``NewType('Derived', Base)`` に対して ``isinstance`` および ``issubclass``、およびサブクラス化は失敗します。関数オブジェクトはこれらの操作をサポートしていないためです。
