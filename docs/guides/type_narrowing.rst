******************************************************************************************
型の絞り込み
******************************************************************************************

Python プログラムには、単一の特定のスコープ内で複数の型を持ち、実行時に条件チェックによって区別されるシンボルが含まれていることがよくあります。 たとえば、次の例では、変数 *name* は ``str`` または ``None`` のいずれかであり、``if name is not None`` はそれを ``str`` のみに絞り込みます::

    def maybe_greet(name: str | None) -> None:
        if name is not None:
            print("Hello, " + name)

この手法は *型の絞り込み* と呼ばれます。
このようなコードで誤検知を避けるために、型チェッカーは Python コードで型を絞り込むために使用されるさまざまな種類の条件チェックを理解します。
型チェッカーが理解する型の絞り込み構文の正確なセットは指定されておらず、型チェッカーによって異なります。 一般的に理解されているパターンには次のものが含まれます。

* ``if x is not None``
* ``if x``
* ``if isinstance(x, SomeType)``
* ``if callable(x)``

ローカル変数の絞り込みに加えて、型チェッカーは通常、インスタンス属性およびシーケンス メンバーの絞り込みもサポートします。たとえば、``if x.some_attribute is not None`` や ``if x[0] is not None`` などです。 ただし、この動作の正確な条件は型チェッカーによって異なります。

サポートされている型の絞り込み構文の詳細については、型チェッカーのドキュメントを参照してください。

型システムには、*ユーザー定義* の型絞り込み関数を作成するための 2 つの方法も含まれています: :py:data:`typing.TypeIs` および :py:data:`typing.TypeGuard`。 これらは、より複雑なチェックを複数の場所で再利用したい場合や、型チェッカーが理解しないチェックを使用する場合に便利です。 このような場合、チェックを実行し、型チェッカーが変数の型を絞り込むために使用できるようにする ``TypeIs`` または ``TypeGuard`` 関数を定義できます。 2 つのうち、``TypeIs`` は通常、より直感的な動作をするため、より多く説明します。 比較については、:ref:`以下 <guide-type-narrowing-typeis-typeguard>` を参照してください。

``TypeIs`` および ``TypeGuard`` の使用方法
------------------------------------------------------------------------------------------

``TypeIs`` 関数は単一の引数を取り、戻り値として ``TypeIs[T]`` を返すように注釈されます。ここで、``T`` は絞り込みたい型です。 関数は、引数が ``T`` 型の場合に ``True`` を返し、それ以外の場合に ``False`` を返す必要があります。 次に、関数は ``isinstance()`` を使用するのと同じように ``if`` チェックで使用できます。 例えば::

    from typing import Literal, TypeIs

    type Direction = Literal["N", "E", "S", "W"]

    def is_direction(x: str) -> TypeIs[Direction]:
        return x in {"N", "E", "S", "W"}

    def maybe_direction(x: str) -> None:
        if is_direction(x):
            print(f"{x} is a cardinal direction")
        else:
            print(f"{x} is not a cardinal direction")

``TypeGuard`` 関数は似たような見た目で、同じ方法で使用されますが、型の絞り込み動作は異なります。 :ref:`以下のセクション <guide-type-narrowing-typeis-typeguard>` で説明されています。

実行している Python のバージョンによっては、標準ライブラリの :py:mod:`typing` モジュールまたはサードパーティの ``typing_extensions`` モジュールから ``TypeIs`` および ``TypeGuard`` をインポートできます。

* ``TypeIs`` は Python 3.13 以降の ``typing`` にあり、バージョン 4.10.0 以降の ``typing_extensions`` にあります。
* ``TypeGuard`` は Python 3.10 以降の ``typing`` にあり、バージョン 3.10.0.0 以降の ``typing_extensions`` にあります。


正しい ``TypeIs`` 関数の作成
------------------------------------------------------------------------------------------

``TypeIs`` 関数を使用すると、型チェッカーの型絞り込み動作を上書きできます。 これは強力なツールですが、誤って記述された ``TypeIs`` 関数は不健全な型チェックにつながる可能性があり、型チェッカーはそのようなエラーを検出できないため、危険です。

``TypeIs[T]`` を返す関数が正しいためには、引数が ``T`` 型の場合にのみ ``True`` を返し、それ以外の場合に ``False`` を返す必要があります。 この条件が満たされない場合、型チェッカーは誤った型を推測する可能性があります。

以下に、正しい ``TypeIs`` 関数と誤った ``TypeIs`` 関数の例を示します::

    from typing import TypeIs

    # 正しい
    def is_int(x: object) -> TypeIs[int]:
        return isinstance(x, int)

    # 誤り: すべての int に対して True を返さない
    def is_positive_int(x: object) -> TypeIs[int]:
        return isinstance(x, int) and x > 0

    # 誤り: 一部の非 int に対して True を返す
    def is_real_number(x: object) -> TypeIs[int]:
        return isinstance(x, (int, float))

この関数は、誤って記述された ``TypeIs`` 関数を使用する場合に発生する可能性のあるエラーを示しています。 これらのエラーは型チェッカーによって検出されません::

    def caller(x: int | str, y: int | float) -> None:
        if is_positive_int(x):  # int に絞り込まれる
            print(x + 1)
        else:  # str に絞り込まれる (誤り)
            print("Hello " + x)  # 実行時エラーが発生する場合がある

        if is_real_number(y):  # int に絞り込まれる
            # 誤った TypeIs のため、y が float の場合、このブランチが実行される。
            print(y.bit_count())  # 実行時エラー: このメソッドは int にのみ存在し、float には存在しない
        else:  # float に絞り込まれる (ただし、実行時には実行されない)
            pass

ここに、より複雑な型の正しい ``TypeIs`` 関数の例を示します::

    from typing import TypedDict, TypeIs

    class Point(TypedDict):
        x: int
        y: int

    def is_point(obj: object) -> TypeIs[Point]:
        return (
            isinstance(obj, dict)
            and all(isinstance(key, str) for key in obj)
            and isinstance(obj.get("x"), int)
            and isinstance(obj.get("y"), int)
        )

.. _`guide-type-narrowing-typeis-typeguard`:

``TypeIs`` および ``TypeGuard``
------------------------------------------------------------------------------------------

:py:data:`typing.TypeIs` および :py:data:`typing.TypeGuard` は、ユーザー定義関数に基づいて変数の型を絞り込むためのツールです。 どちらも引数を取り、入力引数が絞り込まれた型と互換性があるかどうかに応じてブール値を返す関数を注釈するために使用できます。 これらの関数は ``if`` チェックで使用して変数の型を絞り込むことができます。

``TypeIs`` は通常、より直感的な動作をしますが、より多くの制限を導入します。 ``TypeGuard`` は次の場合に使用するのに適したツールです。

* 入力型に :term:`assignable` されていない型に絞り込みたい場合。たとえば、``list[object]`` から ``list[int]`` への絞り込み。 ``TypeIs`` は互換性のある型間でのみ絞り込みを許可します。
* 関数が絞り込まれた型のメンバーであるすべての入力値に対して ``True`` を返さない場合。たとえば、正の整数に対してのみ ``True`` を返す ``TypeGuard[int]`` がある場合。

``TypeIs`` と ``TypeGuard`` は次の点で異なります。

* ``TypeIs`` は、絞り込まれた型が入力型に :term:`assignable` であることを要求しますが、``TypeGuard`` はそうではありません。
* ``TypeGuard`` 関数が ``True`` を返すと、型チェッカーは変数の型を正確に ``TypeGuard`` 型に絞り込みます。 ``TypeIs`` 関数が ``True`` を返すと、型チェッカーは変数の以前に既知の型と ``TypeIs`` 型を組み合わせたより正確な型を推測できます。 (これは「交差型」として知られています。)
* ``TypeGuard`` 関数が ``False`` を返す場合、型チェッカーは変数の型をまったく絞り込むことができません。 ``TypeIs`` 関数が ``False`` を返す場合、型チェッカーは変数の型を ``TypeIs`` 型を除外するように絞り込むことができます。

この動作は次の例で確認できます::

    from typing import TypeGuard, TypeIs, reveal_type, final

    class Base: ...
    class Child(Base): ...
    @final
    class Unrelated: ...

    def is_base_typeguard(x: object) -> TypeGuard[Base]:
        return isinstance(x, Base)

    def is_base_typeis(x: object) -> TypeIs[Base]:
        return isinstance(x, Base)

    def use_typeguard(x: Child | Unrelated) -> None:
        if is_base_typeguard(x):
            reveal_type(x)  # Base
        else:
            reveal_type(x)  # Child | Unrelated

    def use_typeis(x: Child | Unrelated) -> None:
        if is_base_typeis(x):
            reveal_type(x)  # Child
        else:
            reveal_type(x)  # Unrelated


安全性と健全性
------------------------------------------------------------------------------------------

型の絞り込みは、実世界の Python コードの型付けにとって重要ですが、可変性が存在する場合、多くの型の絞り込みは安全ではありません。 型チェッカーは、役立つままで不安全性を最小限に抑える方法で型の絞り込みを制限しようとしますが、すべての安全違反を検出できるわけではありません。

``isinstance()`` および ``issubclass()``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

正確な動作は標準化されていませんが、型チェッカーは通常、``isinstance()`` および ``issubclass()`` への呼び出しに基づいて用語を絞り込むことをサポートします。 ただし、これらの関数には型チェッカーが完全にキャプチャできない複雑なランタイム動作があります。これらの関数は :py:meth:`__instancecheck__` および :py:meth:`__subclasscheck__` 特殊メソッドを呼び出し、これらには任意の複雑なロジックが含まれる場合があります。

これは、これらのメソッドに依存する標準ライブラリの一部に影響を与えます。 :py:class:`abc.ABC` は ``.register()`` メソッドを使用してサブクラスの登録を許可しますが、型チェッカーは通常、このメソッドを認識しません。 :ref:`Runtime-checkable プロトコル <runtime-checkable>` はランタイム ``isinstance()`` チェックをサポートしますが、その動作は型システムと正確に一致しません (たとえば、メソッド パラメーターの型はチェックされません)。

誤った ``TypeIs`` および ``TypeGuard`` 関数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``TypeIs`` と ``TypeGuard`` の両方は、オブジェクトが特定の型であるかどうかを返す関数をユーザーが記述することに依存しています。 ただし、型チェッカーは関数が実際に期待どおりに動作するかどうかを検証しません。 そうでない場合、型チェッカーの絞り込み動作は実行時の動作と一致しません。::

    from typing import TypeIs

    def is_str(x: object) -> TypeIs[str]:
        return True

    def takes_str_or_int(x: str | int) -> None:
        if is_str(x):
            print(x + " is a string")  # 実行時エラー

この問題を回避するために、すべての ``TypeIs`` および ``TypeGuard`` 関数は慎重にレビューおよびテストする必要があります。

不健全な ``TypeGuard`` 絞り込み
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``TypeIs`` とは異なり、``TypeGuard`` は元の型のサブタイプではない型に絞り込むことができます。 これにより、不変データ構造を使用した不安全な動作が可能になります。::

    from typing import Any, TypeGuard

    def is_int_list(x: list[Any]) -> TypeGuard[list[int]]:
        return all(isinstance(i, int) for i in x)

    def maybe_mutate_list(x: list[Any]) -> None:
        if is_int_list(x):
            x.append(0)  # OK, x は list[int] に絞り込まれる

    def takes_bool_list(x: list[bool]) -> None:
        maybe_mutate_list(x)
        reveal_type(x)  # list[bool]
        assert all(isinstance(i, bool) for i in x)  # 実行時に失敗する

    takes_bool_list([True, False])

この問題を回避するために、可能な場合は ``TypeGuard`` の代わりに ``TypeIs`` を使用します。
``TypeGuard`` を使用する必要がある場合は、互換性のない型間での絞り込みを避けてください。
パラメーター注釈で共変の不変型 (例: ``list`` の代わりに ``Sequence`` や ``Iterable``) を使用することをお勧めします。 これを行うと、型絞り込み関数を実装するために ``TypeIs`` を使用できる可能性が高くなります。

無効化された仮定
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

安全性の問題の 1 つのカテゴリは、型の絞り込みがコードのある時点で確立された条件に依存し、その後に依存するという事実に関連しています。最初に ``if x is not None`` をチェックし、その後 ``x`` が ``None`` ではないことに依存します。 ただし、その間に他のコードが実行される場合があります (たとえば、別のスレッド、別のコルーチン、または関数呼び出しによって呼び出されたコード) が、以前の条件を無効にします。

このような問題は、可変オブジェクトの要素に対して絞り込みが行われる場合に最も発生しやすいですが、ローカル変数の絞り込みのみを使用しても安全でない例を構築することは可能です。::

    def maybe_greet(name: str | None) -> None:
        def set_it_to_none():
            nonlocal name
            name = None

        if name is not None:
            set_it_to_none()
            # 実行時に失敗する、現在の型チェッカーではエラーは発生しない
            print("Hello " + name)

    maybe_greet("Guido")

より現実的な例としては、複数のコルーチンがリストを変更する場合があります。::

    import asyncio
    from typing import Sequence, TypeIs

    def is_int_sequence(x: Sequence[object]) -> TypeIs[Sequence[int]]:
        return all(isinstance(i, int) for i in x)

    async def takes_seq(x: Sequence[int | None]):
        if is_int_sequence(x):
            await asyncio.sleep(2)
            print("The total is", sum(x))  # 実行時に失敗する

    async def takes_list(x: list[int | None]):
        t = asyncio.create_task(takes_seq(x))
        await asyncio.sleep(1)
        x.append(None)
        await t

    if __name__ == "__main__":
        lst: list[int | None] = [1, 2, 3]
        asyncio.run(takes_list(lst))

これらの問題は、現在の Python 型システムでは完全には検出できません。 (この問題を解決する別のプログラミング言語の例としては、`所有権 <https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html>`__ というシステムを使用する Rust があります。)
このような問題を回避するには、コードの他の部分から変更されるオブジェクトに対して型の絞り込みを使用しないでください。


関連項目
------------------------------------------------------------------------------------------

* 型の絞り込みに関する型チェッカーのドキュメント

  * `Mypy <https://mypy.readthedocs.io/en/stable/type_narrowing.html>`__
  * `Pyright <https://microsoft.github.io/pyright/#/type-concepts-advanced?id=type-narrowing>`__

* 型の絞り込みに関連する PEP。 これらには、現在の型チェッカーの動作に関する追加の議論と動機が含まれています。

  * :pep:`647` (``TypeGuard`` を導入)
  * (*撤回*) :pep:`724` (``TypeGuard`` の動作の変更を提案)
  * :pep:`742` (``TypeIs`` を導入)
