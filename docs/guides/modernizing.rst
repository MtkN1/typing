.. role:: python(code)
   :language: python

.. role:: t-ext(class)

.. _modernizing:

******************************************************************************************
廃止された型機能のモダナイズ
******************************************************************************************

はじめに
==========================================================================================

このガイドでは、古い型機能を最新のものに置き換えることでコードをモダナイズする方法を説明します。 ここで説明するすべての機能が廃止されているわけではありませんが、より最新の代替機能によって置き換えられており、これらの代替機能の使用が推奨されます。

これらの新しい機能はすべての Python バージョンで利用できるわけではありませんが、一部の機能は `typing-extensions <https://pypi.org/project/typing-extensions/>`_ パッケージからのバックポートとして利用できるか、引用または :python:`from __future__ import annotations` を使用する必要があります。 各セクションでは、機能を使用するために必要な最小 Python バージョン、typing-extensions で利用可能かどうか、および引用を使用して利用可能かどうかを示しています。

.. note::

    typing-extensions の最新バージョンは、サポートが終了していないすべての Python バージョンで利用できますが、古いバージョンでは必ずしも利用できるとは限りません。

.. note::

    :python:`from __future__ import annotations` は Python 3.7 以降で利用可能です。 これは型注釈内でのみ効果があり、引用は依然として外部で必要です。 たとえば、この例は Python 3.7 以降で実行されますが、パイプ演算子は Python 3.10 で導入されました::

        from __future__ import annotations
        from typing_extensions import TypeAlias

        def f(x: int | None) -> int | str: ...  # future import で十分です
        Alias: TypeAlias = "int | str"  # これは引用が必要です

.. _modernizing-type-comments:

型コメント
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.0, 3.6

型コメントは元々、Python 2 および Python 3.6 より前のバージョンで変数注釈をサポートするために導入されました。 ほとんどの型チェッカーは依然としてサポートしていますが、これらは廃止されたと見なされており、型チェッカーはそれらをサポートする必要はありません。

たとえば、次のように置き換えます::

    x = 3  # type: int
    def f(x, y):  # type: (int, int) -> int
        return x + y

次のように置き換えます::

    x: int = 3
    def f(x: int, y: int) -> int:
        return x + y

前方参照や型チェック時にのみ利用可能な型を使用する場合は、:python:`from __future__ import annotations` (Python 3.7 以降で利用可能) を使用するか、型を引用する必要があります::

    def f(x: "Parrot") -> int: ...

    class Parrot: ...

.. _modernizing-typing-text:

``typing.Text``
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.0

:class:`typing.Text` は Python 2 互換性のための型エイリアスでした。 これは :class:`str` と同等であり、置き換える必要があります。
たとえば、次のように置き換えます::

    from typing import Text

    def f(x: Text) -> Text: ...

次のように置き換えます::

    def f(x: str) -> str: ...

.. _modernizing-typed-dict:

``typing.TypedDict`` レガシーフォーム
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.6

:class:`TypedDict <typing.TypedDict>` は、変数注釈をサポートしない Python バージョンをサポートするための 2 つのレガシーフォームをサポートしています。 次の 2 つのバリアントを置き換えます::

    from typing import TypedDict

    FlyingSaucer = TypedDict("FlyingSaucer", {"x": int, "y": str})
    FlyingSaucer = TypedDict("FlyingSaucer", x=int, y=str)

次のように置き換えます::

    class FlyingSaucer(TypedDict):
        x: int
        y: str

ただし、キーが有効な Python 識別子でない場合は、辞書形式が依然として必要です::

    Airspeeds = TypedDict("Airspeeds", {"unladen-swallow": int})

.. _modernizing-generics:

``typing`` モジュールのジェネリクス
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.0 (引用), Python 3.9 (非引用)

元々、:mod:`typing` モジュールは、型パラメーターを受け入れる組み込み型のエイリアスを提供していました。 Python 3.9 以降、これらのエイリアスは不要になり、組み込み型に置き換えることができます。 たとえば、次のように置き換えます::

    from typing import Dict, List

    def f(x: List[int]) -> Dict[str, int]: ...

次のように置き換えます::

    def f(x: list[int]) -> dict[str, int]: ...

これには次の型が影響します:

* :class:`typing.Dict` (→ :class:`dict`)
* :class:`typing.FrozenSet` (→ :class:`frozenset`)
* :class:`typing.List` (→ :class:`list`)
* :class:`typing.Set` (→ :class:`set`)
* :data:`typing.Tuple` (→ :class:`tuple`)

:mod:`typing` モジュールは、型パラメーターを受け入れる特定の標準ライブラリ型のエイリアスも提供していました。 Python 3.9 以降、これらのエイリアスは不要になり、適切な型に置き換えることができます。 たとえば、次のように置き換えます::

    from typing import DefaultDict, Pattern

    def f(x: Pattern[str]) -> DefaultDict[str, int]: ...

次のように置き換えます::

    from collections import defaultdict
    from re import Pattern

    def f(x: Pattern[str]) -> defaultdict[str, int]: ...

これには次の型が影響します:

* :class:`typing.Deque` (→ :class:`collections.deque`)
* :class:`typing.DefaultDict` (→ :class:`collections.defaultdict`)
* :class:`typing.OrderedDict` (→ :class:`collections.OrderedDict`)
* :class:`typing.Counter` (→ :class:`collections.Counter`)
* :class:`typing.ChainMap` (→ :class:`collections.ChainMap`)
* :class:`typing.Awaitable` (→ :class:`collections.abc.Awaitable`)
* :class:`typing.Coroutine` (→ :class:`collections.abc.Coroutine`)
* :class:`typing.AsyncIterable` (→ :class:`collections.abc.AsyncIterable`)
* :class:`typing.AsyncIterator` (→ :class:`collections.abc.AsyncIterator`)
* :class:`typing.AsyncGenerator` (→ :class:`collections.abc.AsyncGenerator`)
* :class:`typing.Iterable` (→ :class:`collections.abc.Iterable`)
* :class:`typing.Iterator` (→ :class:`collections.abc.Iterator`)
* :class:`typing.Generator` (→ :class:`collections.abc.Generator`)
* :class:`typing.Reversible` (→ :class:`collections.abc.Reversible`)
* :class:`typing.Container` (→ :class:`collections.abc.Container`)
* :class:`typing.Collection` (→ :class:`collections.abc.Collection`)
* :data:`typing.Callable` (→ :class:`collections.abc.Callable`)
* :class:`typing.AbstractSet` (→ :class:`collections.abc.Set`), 名前の変更に注意
* :class:`typing.MutableSet` (→ :class:`collections.abc.MutableSet`)
* :class:`typing.Mapping` (→ :class:`collections.abc.Mapping`)
* :class:`typing.MutableMapping` (→ :class:`collections.abc.MutableMapping`)
* :class:`typing.Sequence` (→ :class:`collections.abc.Sequence`)
* :class:`typing.MutableSequence` (→ :class:`collections.abc.MutableSequence`)
* :class:`typing.ByteString` (→ :class:`collections.abc.ByteString`), ただし :ref:`modernizing-byte-string` を参照
* :class:`typing.MappingView` (→ :class:`collections.abc.MappingView`)
* :class:`typing.KeysView` (→ :class:`collections.abc.KeysView`)
* :class:`typing.ItemsView` (→ :class:`collections.abc.ItemsView`)
* :class:`typing.ValuesView` (→ :class:`collections.abc.ValuesView`)
* :class:`typing.ContextManager` (→ :class:`contextlib.AbstractContextManager`), 名前の変更に注意
* :class:`typing.AsyncContextManager` (→ :class:`contextlib.AbstractAsyncContextManager`), 名前の変更に注意
* :class:`typing.Pattern` (→ :class:`re.Pattern`)
* :class:`typing.Match` (→ :class:`re.Match`)

.. _modernizing-union:

``typing.Union`` および ``typing.Optional``
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.0 (引用), Python 3.10 (非引用)

:data:`Union <typing.Union>` および :data:`Optional <typing.Optional>` は廃止されたとは見なされていませんが、``|`` (パイプ) 演算子を使用する方が読みやすい場合がよくあります。 :python:`Union[X, Y]` は :python:`X | Y` と同等であり、:python:`Optional[X]` は :python:`X | None` と同等です。

たとえば、次のように置き換えます::

    from typing import Optional, Union

    def f(x: Optional[int]) -> Union[int, str]: ...

次のように置き換えます::

    def f(x: int | None) -> int | str: ...

.. _modernizing-no-return:

``typing.NoReturn``
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.11, typing-extensions

Python 3.11 では、:data:`typing.NoReturn` のエイリアスとして :data:`typing.Never` が導入され、戻り値の型ではない注釈で使用されます。 たとえば、次のように置き換えます::

    from typing import NoReturn

    def f(x: int, y: NoReturn) -> None: ...

次のように置き換えます::

    from typing import Never  # または typing_extensions.Never

    def f(x: int, y: Never) -> None: ...

ただし、戻り値の型には ``NoReturn`` を使用します::

    from typing import NoReturn

    def f(x: int) -> NoReturn: ...

.. _modernizing-type-aliases:

型エイリアス
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.12 (キーワード); Python 3.10, typing-extensions

元々、型エイリアスは単純な代入を使用して定義されていました::

    IntList = list[int]

Python 3.12 では、型エイリアスを定義するための :keyword:`type` キーワードが導入されました::

    type IntList = list[int]

古い Python バージョンをサポートするコードは、Python 3.10 で導入されたが typing-extensions でも利用可能な :data:`TypeAlias <typing.TypeAlias>` を代わりに使用する必要があります::

    from typing import TypeAlias  # または typing_extensions.TypeAlias

    IntList: TypeAlias = list[int]

.. _modernizing-user-generics:

ユーザー定義ジェネリクス
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.12

Python 3.12 では、ジェネリック クラスを定義するための新しい構文が導入されました。 以前は、ジェネリック クラスは :class:`typing.Generic` (または他のジェネリック クラス) から派生し、型変数は :class:`typing.TypeVar` を使用して定義されていました。 たとえば::

    from typing import Generic, TypeVar

    T = TypeVar("T")

    class Brian(Generic[T]): ...
    class Reg(int, Generic[T]): ...

Python 3.12 以降、型変数は ``TypeVar`` を使用して宣言する必要がなくなり、クラスを ``Generic`` から派生させる代わりに次の構文を使用できます::

    class Brian[T]: ...
    class Reg[T](int): ...

.. _modernizing-byte-string:

``typing.ByteString``
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.0; Python 3.12, typing-extensions

:class:`ByteString <typing.ByteString>` は元々、:class:`bytes`、:class:`bytearray`、および :class:`memoryview` のような「バイト型」の型のエイリアスとして意図されていました。 実際には、これはほとんどの場合、正確に必要なものではありません。 代わりに次のいずれかの代替案を使用します:

* 公開 API を宣言しない場合は、単に :class:`bytes` で十分な場合がよくあります。
* :ref:`buffer protocol <bufferobjects>` をサポートする任意の型を受け入れる項目には、:class:`collections.abc.Buffer` (Python 3.12 以降で利用可能) または :t-ext:`typing_extensions.Buffer` を使用します。
* それ以外の場合は、:class:`bytes`、:class:`bytearray`、:class:`memoryview`、および/または受け入れられる他の型の共用体を使用します。

``typing.Hashable`` および ``typing.Sized``
==========================================================================================

*代替機能が利用可能になったのは:* Python 3.12, typing-extensions

次の :mod:`typing` の抽象基本クラスは、Python 3.12 で :mod:`collections.abc` に追加されました:

* :class:`typing.Hashable` (→ :class:`collections.abc.Hashable`)
* :class:`typing.Sized` (→ :class:`collections.abc.Sized`)

インポートを新しい場所に更新します::

    from collections.abc import Hashable, Sized

    def f(x: Hashable) -> Sized: ...
