.. _`overload`:

``@overload``
==========================================================================================

``@overload`` デコレータは、引数の型の異なる複数の組み合わせをサポートする関数やメソッドを記述することを可能にします。 このパターンは、組み込みモジュールや型で頻繁に使用されます。 例えば、``bytes`` 型の ``__getitem__()`` メソッドは次のように記述できます::

  from typing import overload

  class bytes:
      ...
      @overload
      def __getitem__(self, i: int) -> int: ...
      @overload
      def __getitem__(self, s: slice) -> bytes: ...

この記述は、引数と戻り値の型の関係を表現できないユニオンを使用するよりも正確です::

  class bytes:
      ...
      def __getitem__(self, a: int | slice) -> int | bytes: ...

``@overload`` が便利なもう一つの例は、呼び出し可能なオブジェクトの型に応じて異なる数の引数を取る組み込みの ``map()`` 関数の型です::

  from typing import TypeVar, overload
  from collections.abc import Callable, Iterable, Iterator

  T1 = TypeVar('T1')
  T2 = TypeVar('T2')
  S = TypeVar('S')

  @overload
  def map(func: Callable[[T1], S], iter1: Iterable[T1]) -> Iterator[S]: ...
  @overload
  def map(func: Callable[[T1, T2], S],
          iter1: Iterable[T1], iter2: Iterable[T2]) -> Iterator[S]: ...
  # ... and we could add more items to support more than two iterables

次のようにして ``map(None, ...)`` をサポートする項目を簡単に追加することもできます::

  @overload
  def map(func: None, iter1: Iterable[T1]) -> Iterable[T1]: ...
  @overload
  def map(func: None,
          iter1: Iterable[T1],
          iter2: Iterable[T2]) -> Iterable[tuple[T1, T2]]: ...

上記のように使用される ``@overload`` デコレータは、スタブファイルに適しています。 通常のモジュールでは、一連の ``@overload`` デコレータが付けられた定義の後には、必ず1つの ``@overload`` デコレータが付けられていない定義（同じ関数/メソッドのため）が続かなければなりません。 ``@overload`` デコレータが付けられた定義は型チェッカーのためだけのものであり、非 ``@overload`` デコレータが付けられた定義によって上書きされます。後者は実行時に使用されますが、型チェッカーによって無視されるべきです。 実行時に ``@overload`` デコレータが付けられた関数を直接呼び出すと ``NotImplementedError`` が発生します。 ここに、ユニオンや型変数を使用して簡単に表現できない非スタブのオーバーロードの例があります::

  @overload
  def utf8(value: None) -> None:
      pass
  @overload
  def utf8(value: bytes) -> bytes:
      pass
  @overload
  def utf8(value: unicode) -> bytes:
      pass
  def utf8(value):
      <actual implementation>

制約された ``TypeVar`` 型は、``@overload`` デコレータを使用する代わりに使用されることがよくあります。 例えば、このスタブファイルの ``concat1`` と ``concat2`` の定義は同等です::

  from typing import TypeVar

  AnyStr = TypeVar('AnyStr', str, bytes)

  def concat1(x: AnyStr, y: AnyStr) -> AnyStr: ...

  @overload
  def concat2(x: str, y: str) -> str: ...
  @overload
  def concat2(x: bytes, y: bytes) -> bytes: ...

上記の ``map`` や ``bytes.__getitem__`` のように、型変数を使用して正確に表現できない関数もあります。 型変数が十分でない場合にのみ ``@overload`` を使用することをお勧めします。

``AnyStr`` のような型変数と ``@overload`` を使用することのもう一つの重要な違いは、前者はジェネリッククラスの型パラメータの制約を定義するためにも使用できることです。 例えば、ジェネリッククラス ``typing.IO`` の型パラメータは制約されています（``IO[str]``、``IO[bytes]``、``IO[Any]`` のみが有効です）::

  class IO(Generic[AnyStr]): ...
