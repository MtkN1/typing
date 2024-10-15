.. _protocol-types:

プロトコルと構造的サブタイピング
==========================================================================================

Python の型システムは、2 つのオブジェクトが型として互換性があるかどうかを判断する 2 つの方法をサポートしています: 名目上のサブタイピングと構造的サブタイピング。

*名目上の* サブタイピングは、クラス階層に厳密に基づいています。 クラス ``Dog`` がクラス ``Animal`` を継承する場合、それは ``Animal`` のサブタイプです。 ``Dog`` のインスタンスは、``Animal`` のインスタンスが期待される場合に使用できます。 この形式のサブタイピングは、Python の型システムが主に使用するものです: 理解しやすく、明確で簡潔なエラーメッセージを生成し、クラス階層に基づいて動作するネイティブの :py:func:`isinstance <isinstance>` チェックと一致します。

*構造的* サブタイピングは、オブジェクトで実行できる操作に基づいています。 クラス ``Dog`` は、後者のすべての属性とメソッドを持ち、互換性のある型を持つ場合、クラス ``Animal`` の構造的サブタイプです。

構造的サブタイピングは、Python プログラマーによく知られているダックタイピングの静的な同等物と見なすことができます。 プロトコルと構造的サブタイピングの詳細な仕様については、:pep:`544` を参照してください。

.. _predefined_protocols:

定義済みプロトコル
******************************************************************************************

:py:mod:`typing` モジュールは、:py:class:`Iterable[T] <typing.Iterable>` など、Python でダックタイピングが一般的に使用される場所に対応するさまざまなプロトコル クラスを定義します。

クラスが適切な :py:meth:`__iter__ <object.__iter__>` メソッドを定義している場合、型チェッカーはそれが :py:term:`iterable` プロトコルを実装し、:py:class:`Iterable[T] <typing.Iterable>` と互換性があることを理解します。 例えば、以下の ``IntList`` は ``int`` 値を反復処理できます:

.. code-block:: python

   from typing import Iterator, Iterable, Optional

   class IntList:
       def __init__(self, value: int, next_node: Optional['IntList']) -> None:
           self.value = value
           self.next_node = next_node

       def __iter__(self) -> Iterator[int]:
           current = self
           while current:
               yield current.value
               current = current.next_node

   def print_numbered(items: Iterable[int]) -> None:
       for n, x in enumerate(items):
           print(n + 1, x)

   x = IntList(3, IntList(5, None))
   print_numbered(x)  # OK
   print_numbered([4, 5])  # Also OK

:ref:`predefined_protocols_reference` には、:py:mod:`typing` で定義されているすべてのプロトコルと、それぞれのプロトコルを実装するために定義する必要がある対応するメソッドのシグネチャが記載されています。

シンプルなユーザー定義プロトコル
******************************************************************************************

特別な ``Protocol`` クラスを継承することで、独自のプロトコル クラスを定義できます:

.. code-block:: python

   from typing import Iterable
   from typing_extensions import Protocol

   class SupportsClose(Protocol):
       # 空のメソッド本体 (明示的な '...')
       def close(self) -> None: ...

   class Resource:  # SupportsClose 基本クラスはありません!

       def close(self) -> None:
           self.resource.release()

       # ... 他のメソッド ...

   def close_all(items: Iterable[SupportsClose]) -> None:
       for item in items:
           item.close()

   close_all([Resource(), open('some/file')])  # OK

``Resource`` は、互換性のある ``close`` メソッドを定義しているため、``SupportsClose`` プロトコルのサブタイプです。 :py:func:`open` によって返される通常のファイル オブジェクトも同様にプロトコルと互換性があります。これらは ``close()`` をサポートしています。

サブプロトコルの定義とプロトコルのサブクラス化
******************************************************************************************

サブプロトコルを定義することもできます。 既存のプロトコルは、複数の継承を使用して拡張およびマージできます。 例:

.. code-block:: python

   # ... 前の例から続く

   class SupportsRead(Protocol):
       def read(self, amount: int) -> bytes: ...

   class TaggedReadableResource(SupportsClose, SupportsRead, Protocol):
       label: str

   class AdvancedResource(Resource):
       def __init__(self, label: str) -> None:
           self.label = label

       def read(self, amount: int) -> bytes:
           # いくつかの実装
           ...

   resource: TaggedReadableResource
   resource = AdvancedResource('handle with care')  # OK

既存のプロトコルを継承しても、サブクラスが自動的にプロトコルになるわけではないことに注意してください。 それは、与えられたプロトコル (またはプロトコル) を実装する通常の (非プロトコル) クラスまたは ABC を作成するだけです。 プロトコルを定義する場合、``Protocol`` 基本クラスは常に明示的に存在する必要があります:

.. code-block:: python

   class NotAProtocol(SupportsClose):  # これはプロトコルではありません
       new_attr: int

   class Concrete:
       new_attr: int = 0

       def close(self) -> None:
           ...

   # エラー: デフォルトで名目上のサブタイピングが使用されます
   x: NotAProtocol = Concrete()  # エラー!

プロトコルにはメソッドのデフォルト実装を含めることもできます。 これらのプロトコルを明示的にサブクラス化すると、これらのデフォルト実装が継承されます。

プロトコルを基本クラスとして明示的に含めることは、クラスが特定のプロトコルを実装していることを文書化する方法でもあり、クラス実装が実際にプロトコルと互換性があることを型チェッカーに強制する方法でもあります。 特に、属性の値やメソッド本体を省略すると、それが暗黙的に抽象的であると見なされます:

.. code-block:: python

   class SomeProto(Protocol):
       attr: int  # 注意、右辺はありません
       # 本文が文字通り "..." のみである場合、明示的なサブクラスはメソッドを実装しない限り抽象クラスとして扱われます。
       def method(self) -> str: ...

   class ExplicitSubclass(SomeProto):
       pass

   ExplicitSubclass()  # エラー: 抽象属性 'attr' と 'method' を持つ抽象クラス 'ExplicitSubclass' をインスタンス化できません

同様に、プロトコル インスタンスに明示的に代入することは、クラスがプロトコルを実装していることを型チェッカーに確認させる方法です:

.. code-block:: python

   _proto: SomeProto = cast(ExplicitSubclass, None)

プロトコル属性の不変性
******************************************************************************************

プロトコルに関する一般的な問題は、プロトコル属性が不変であることです。 例えば:

.. code-block:: python

   class Box(Protocol):
        content: object

   class IntBox:
        content: int

   def takes_box(box: Box) -> None: ...

   takes_box(IntBox())  # エラー: "takes_box" への引数 1 の型が一致しません。予期される型は "IntBox" です。 "Box" が予期されます
                        # 注: "IntBox" の次のメンバーに競合があります:
                        # 注:      content: 予期される型 "object"、取得された型 "int"

これは、``Box`` が ``content`` を可変属性として定義しているためです。 これが問題となる理由は次のとおりです:

.. code-block:: python

   def takes_box_evil(box: Box) -> None:
       box.content = "asdf"  # これは悪いです。box.content はオブジェクトである必要があります

   my_int_box = IntBox()
   takes_box_evil(my_int_box)
   my_int_box.content + 1  # おっと、TypeError!

これは、``Box`` プロトコルで ``@property`` を使用して ``content`` を読み取り専用として宣言することで修正できます:

.. code-block:: python

   class Box(Protocol):
       @property
       def content(self) -> object: ...

   class IntBox:
       content: int

   def takes_box(box: Box) -> None: ...

   takes_box(IntBox(42))  # OK

再帰プロトコル
******************************************************************************************

プロトコルは再帰的 (自己参照) および相互再帰的にすることができます。 これは、ツリーやリンク リストなどの抽象再帰コレクションを宣言するのに役立ちます:

.. code-block:: python

   from typing import TypeVar, Optional
   from typing_extensions import Protocol

   class TreeLike(Protocol):
       value: int

       @property
       def left(self) -> Optional['TreeLike']: ...

       @property
       def right(self) -> Optional['TreeLike']: ...

   class SimpleTree:
       def __init__(self, value: int) -> None:
           self.value = value
           self.left: Optional['SimpleTree'] = None
           self.right: Optional['SimpleTree'] = None

   root: TreeLike = SimpleTree(0)  # OK

プロトコルで isinstance() を使用する
******************************************************************************************

プロトコル クラスを :py:func:`isinstance` で使用する場合は、``@runtime_checkable`` クラス デコレータで装飾します。 デコレータは、基本的なランタイム構造チェックのサポートを追加します:

.. code-block:: python

   from typing_extensions import Protocol, runtime_checkable

   @runtime_checkable
   class Portable(Protocol):
       handles: int

   class Mug:
       def __init__(self) -> None:
           self.handles = 1

   def use(handles: int) -> None: ...

   mug = Mug()
   if isinstance(mug, Portable):  # ランタイムで動作します!
       use(mug.handles)

:py:func:`isinstance` は、:py:mod:`typing` の :ref:`predefined protocols <predefined_protocols>` でも機能します。 例えば :py:class:`~typing.Iterable`。

.. warning::
   :py:func:`isinstance` をプロトコルで使用することは、ランタイムでは完全に安全ではありません。
   例えば、メソッドのシグネチャはチェックされません。 ランタイム実装は、すべてのプロトコル メンバーが存在することのみをチェックし、
   正しい型を持っていることはチェックしません。 プロトコルで :py:func:`issubclass` を使用すると、メソッドの存在のみがチェックされます。

.. note::
   プロトコルで :py:func:`isinstance` を使用すると、驚くほど遅くなることがあります。
   多くの場合、属性の存在を確認するために :py:func:`hasattr` を使用する方が適しています。

.. _callback_protocols:

コールバック プロトコル
******************************************************************************************

プロトコルを使用して、可変長、オーバーロード、および複雑なジェネリック コールバックなど、:py:data:`Callable[...] <typing.Callable>` 構文を使用して表現するのが難しい (または不可能な) 柔軟なコールバック型を定義できます。 これらは、特別な :py:meth:`__call__ <object.__call__>` メンバーで定義されます:

.. code-block:: python

   from typing import Optional, Iterable
   from typing_extensions import Protocol

   class Combiner(Protocol):
       def __call__(self, *vals: bytes, maxlen: Optional[int] = None) -> list[bytes]: ...

   def batch_proc(data: Iterable[bytes], cb_results: Combiner) -> bytes:
       for item in data:
           ...

   def good_cb(*vals: bytes, maxlen: Optional[int] = None) -> list[bytes]:
       ...
   def bad_cb(*vals: bytes, maxitems: Optional[int]) -> list[bytes]:
       ...

   batch_proc([], good_cb)  # OK
   batch_proc([], bad_cb)   # エラー! 引数 2 の型が一致しません。コールバックの名前と種類が異なります

コールバック プロトコルと :py:data:`~typing.Callable` 型は、ほとんどの場合、互換性があります。
:py:meth:`__call__ <object.__call__>` メソッドの引数名は同一でなければなりませんが、ダブル アンダースコアのプレフィックスが使用されている場合は例外です。 例えば:

.. code-block:: python

   from typing import Callable, TypeVar
   from typing_extensions import Protocol

   T = TypeVar('T')

   class Copy(Protocol):
       def __call__(self, __origin: T) -> T: ...

   copy_a: Callable[[T], T]
   copy_b: Copy

   copy_a = copy_b  # OK
   copy_b = copy_a  # これも OK

.. _predefined_protocols_reference:

定義済みプロトコル リファレンス
******************************************************************************************

反復プロトコル
------------------------------------------------------------------------------------------

反復プロトコルを使用すると、``for`` ループ内で反復処理できるものや、:py:func:`next` に渡すことができるものを型指定できます。

Iterable[T]
-----------

:ref:`example above <predefined_protocols>` には、シンプルな :py:meth:`__iter__ <object.__iter__>` メソッドの実装があります。

.. code-block:: python

   def __iter__(self) -> Iterator[T]

:py:class:`~typing.Iterable` も参照してください。

Iterator[T]
-----------

.. code-block:: python

   def __next__(self) -> T
   def __iter__(self) -> Iterator[T]

:py:class:`~typing.Iterator` も参照してください。

コレクション プロトコル
------------------------------------------------------------------------------------------

これらの多くは、:py:class:`list` や :py:class:`dict` などの組み込みコンテナー型によって実装されており、ユーザー定義のコレクション オブジェクトにも役立ちます。

Sized
-----

これは、:py:func:`len(x) <len>` をサポートするオブジェクトの型です。

.. code-block:: python

   def __len__(self) -> int

:py:class:`~typing.Sized` も参照してください。

Container[T]
------------

これは、``in`` 演算子をサポートするオブジェクトの型です。

.. code-block:: python

   def __contains__(self, x: object) -> bool

:py:class:`~typing.Container` も参照してください。

Collection[T]
-------------

.. code-block:: python

   def __len__(self) -> int
   def __iter__(self) -> Iterator[T]
   def __contains__(self, x: object) -> bool

:py:class:`~typing.Collection` も参照してください。

一度限りのプロトコル
------------------------------------------------------------------------------------------

これらのプロトコルは、通常、単一の標準ライブラリ関数またはクラスでのみ役立ちます。

Reversible[T]
-------------

これは、:py:func:`reversed(x) <reversed>` をサポートするオブジェクトの型です。

.. code-block:: python

   def __reversed__(self) -> Iterator[T]

:py:class:`~typing.Reversible` も参照してください。

SupportsAbs[T]
--------------

これは、:py:func:`abs(x) <abs>` をサポートするオブジェクトの型です。 ``T`` は :py:func:`abs(x) <abs>` によって返される値の型です。

.. code-block:: python

   def __abs__(self) -> T

:py:class:`~typing.SupportsAbs` も参照してください。

SupportsBytes
-------------

これは、:py:class:`bytes(x) <bytes>` をサポートするオブジェクトの型です。

.. code-block:: python

   def __bytes__(self) -> bytes

:py:class:`~typing.SupportsBytes` も参照してください。

.. _supports-int-etc:

SupportsComplex
---------------

これは、:py:class:`complex(x) <complex>` をサポートするオブジェクトの型です。 算術演算はサポートされていないことに注意してください。

.. code-block:: python

   def __complex__(self) -> complex

:py:class:`~typing.SupportsComplex` も参照してください。

SupportsFloat
-------------

これは、:py:class:`float(x) <float>` をサポートするオブジェクトの型です。 算術演算はサポートされていないことに注意してください。

.. code-block:: python

   def __float__(self) -> float

:py:class:`~typing.SupportsFloat` も参照してください。

SupportsInt
-----------

これは、:py:class:`int(x) <int>` をサポートするオブジェクトの型です。 算術演算はサポートされていないことに注意してください。

.. code-block:: python

   def __int__(self) -> int

:py:class:`~typing.SupportsInt` も参照してください。

SupportsRound[T]
----------------

これは、:py:func:`round(x) <round>` をサポートするオブジェクトの型です。

.. code-block:: python

   def __round__(self) -> T

:py:class:`~typing.SupportsRound` も参照してください。

非同期プロトコル
------------------------------------------------------------------------------------------

これらのプロトコルは、非同期コードで役立ちます。

Awaitable[T]
------------

.. code-block:: python

   def __await__(self) -> Generator[Any, None, T]

:py:class:`~typing.Awaitable` も参照してください。

AsyncIterable[T]
----------------

.. code-block:: python

   def __aiter__(self) -> AsyncIterator[T]

:py:class:`~typing.AsyncIterable` も参照してください。

AsyncIterator[T]
----------------

.. code-block:: python

   def __anext__(self) -> Awaitable[T]
   def __aiter__(self) -> AsyncIterator[T]

:py:class:`~typing.AsyncIterator` も参照してください。

コンテキスト マネージャ プロトコル
------------------------------------------------------------------------------------------

コンテキスト マネージャには 2 つのプロトコルがあります。1 つは通常のコンテキスト マネージャ用、もう 1 つは非同期コンテキスト マネージャ用です。 これらを使用すると、``with`` および ``async with`` ステートメントで使用できるオブジェクトを定義できます。

ContextManager[T]
-----------------

.. code-block:: python

   def __enter__(self) -> T
   def __exit__(self,
                exc_type: Optional[Type[BaseException]],
                exc_value: Optional[BaseException],
                traceback: Optional[TracebackType]) -> Optional[bool]

:py:class:`~typing.ContextManager` も参照してください。

AsyncContextManager[T]
----------------------

.. code-block:: python

   def __aenter__(self) -> Awaitable[T]
   def __aexit__(self,
                 exc_type: Optional[Type[BaseException]],
                 exc_value: Optional[BaseException],
                 traceback: Optional[TracebackType]) -> Awaitable[Optional[bool]]

:py:class:`~typing.AsyncContextManager` も参照してください。

クレジット
******************************************************************************************

このドキュメントは、`mypy documentation <https://mypy.readthedocs.io/en/stable/>`_ に基づいています。
