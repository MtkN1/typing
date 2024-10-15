.. _`type-narrowing`:

型の絞り込み
==========================================================================================

型チェッカーは特定のコンテキストで式の型を絞り込むべきです。 この動作は現在ほとんど指定されていません。

.. _`typeguard`:

TypeGuard
------------------------------------------------------------------------------------------

（元々 :pep:`647` で指定されています。）

``TypeGuard`` シンボルは ``typing`` モジュールからエクスポートされる :term:`special form` であり、単一の型引数を受け取ります。 これはユーザー定義の型ガード関数の戻り値の型を注釈するために使用されます。 型ガード関数内の return 文は bool 値を返すべきであり、型チェッカーはすべての return パスが bool を返すことを検証するべきです。

``TypeGuard`` はコールバックプロトコルや ``Callable`` :term:`special form` などの戻り値の型としても有効です。 これらのコンテキストでは、bool のサブタイプとして扱われます。 例えば、``Callable[..., TypeGuard[int]]`` は ``Callable[..., bool]`` に代入可能です。

``TypeGuard`` が少なくとも 1 つのパラメータを受け取る関数またはメソッドの戻り値の型を注釈するために使用される場合、その関数またはメソッドは型チェッカーによってユーザー定義の型ガードとして扱われます。 ``TypeGuard`` に提供される型引数は、関数によって検証された型を示します。

ユーザー定義の型ガードは、次の例のようにジェネリック関数にすることができます。

::

    _T = TypeVar("_T")

    def is_two_element_tuple(val: Tuple[_T, ...]) -> TypeGuard[tuple[_T, _T]]:
        return len(val) == 2

    def func(names: tuple[str, ...]):
        if is_two_element_tuple(names):
            reveal_type(names)  # tuple[str, str]
        else:
            reveal_type(names)  # tuple[str, ...]

型チェッカーは、ユーザー定義の型ガードに最初の位置引数として渡される式に対して型の絞り込みを適用するべきです。 型ガード関数が複数の引数を受け取る場合、それらの追加の引数式には型の絞り込みは適用されません。

型ガード関数がインスタンスメソッドまたはクラスメソッドとして実装されている場合、最初の位置引数は 2 番目のパラメータ（「self」または「cls」）にマップされます。

次に、複数の引数を受け取るユーザー定義の型ガード関数の例を示します。

::

    def is_str_list(val: list[object], allow_empty: bool) -> TypeGuard[list[str]]:
        if len(val) == 0:
            return allow_empty
        return all(isinstance(x, str) for x in val)

    _T = TypeVar("_T")

    def is_set_of(val: set[Any], type: type[_T]) -> TypeGuard[Set[_T]]:
        return all(isinstance(x, type) for x in val)

ユーザー定義の型ガード関数の戻り値の型は通常、最初の引数の型よりも厳密に「狭い」型を参照します（つまり、より一般的な型に代入できるより具体的な型です）。 ただし、戻り値の型が厳密に狭い必要はありません。 これにより、上記の例のように ``list[str]`` が ``list[object]`` に代入できない場合などのケースが許容されます。

条件文にユーザー定義の型ガード関数への呼び出しが含まれており、その関数が true を返す場合、型チェッカーは型ガード関数に最初の位置引数として渡された式が TypeGuard の戻り値の型で指定された型を持つと仮定するべきです。 ただし、条件コードブロック内でさらに絞り込まれるまでの間です。

一部の組み込み型ガードは、正のテストと負のテストの両方に対して絞り込みを提供します（``if`` および ``else`` 節の両方）。 例えば、``x is None`` 形式の式の型ガードを考えてみましょう。 ``x`` が None と他の型のユニオン型を持っている場合、正の場合には ``None`` に絞り込まれ、負の場合には他の型に絞り込まれます。 ユーザー定義の型ガードは正の場合（``if`` 節）にのみ絞り込みを適用します。 負の場合には型は絞り込まれません。

::

    OneOrTwoStrs = tuple[str] | tuple[str, str]
    def func(val: OneOrTwoStrs):
        if is_two_element_tuple(val):
            reveal_type(val)  # tuple[str, str]
            ...
        else:
            reveal_type(val)   # OneOrTwoStrs
            ...

        if not is_two_element_tuple(val):
            reveal_type(val)   # OneOrTwoStrs
            ...
        else:
            reveal_type(val)  # tuple[str, str]
            ...

.. _`typeis`:

TypeIs
------------------------------------------------------------------------------------------

（元々 :pep:`742` で指定されています。）

:term:`special form` ``TypeIs`` は、使用方法、動作、およびランタイム実装が ``TypeGuard`` と類似しています。

``TypeIs`` は単一の型引数を受け取り、関数の戻り値の型として使用できます。 ``TypeIs`` を返すように注釈された関数は「型絞り込み関数」と呼ばれます。 型絞り込み関数は ``bool`` 値を返す必要があり、型チェッカーはすべての return パスが ``bool`` を返すことを検証するべきです。

型絞り込み関数は少なくとも 1 つの位置引数を受け取る必要があります。 型絞り込み動作は関数に渡される最初の位置引数に適用されます。 関数は追加の引数を受け取ることができますが、それらは型絞り込みの影響を受けません。 型絞り込み関数がインスタンスメソッドまたはクラスメソッドとして実装されている場合、最初の位置引数は 2 番目のパラメータ（``self`` または ``cls`` の後）にマップされます。

``TypeIs`` の動作を指定するために、次の用語を使用します。

* I = ``TypeIs`` 入力型
* R = ``TypeIs`` 戻り値の型
* A = 型絞り込み関数に渡される引数の型（絞り込み前）
* NP = 絞り込まれた型（正の場合；``TypeIs`` が ``True`` を返した場合に使用）
* NN = 絞り込まれた型（負の場合；``TypeIs`` が ``False`` を返した場合に使用）

  ::

    def narrower(x: I) -> TypeIs[R]: ...

    def func1(val: A):
        if narrower(val):
            assert_type(val, NP)
        else:
            assert_type(val, NN)

戻り値の型 ``R`` は ``I`` に :term:`assignable` でなければなりません。 この条件が満たされない場合、型チェッカーはエラーを出すべきです。

形式的には、型 *NP* は :math:`A \land R` に絞り込まれるべきであり、型 *NN* は :math:`A \land \neg R` に絞り込まれるべきです。 実際には、厳密な型ガードの理論的な型は Python 型システムで正確に表現することはできません。 型チェッカーはこれらの型の実用的な近似に頼るべきです。 経験則として、型チェッカーは :py:func:`isinstance` の処理と同じ型絞り込みロジックを使用し、一貫した結果を得るべきです。 このガイダンスにより、将来的に型システムが拡張された場合の変更や改善が許容されます。

型絞り込みは正の場合と負の場合の両方に適用されます::

    from typing import TypeIs, assert_type

    def is_str(x: object) -> TypeIs[str]:
        return isinstance(x, str)

    def f(x: str | int) -> None:
        if is_str(x):
            assert_type(x, str)
        else:
            assert_type(x, int)

最終的に絞り込まれた型は、引数の既知の型の制約により **R** よりも狭くなる場合があります::

    from collections.abc import Awaitable
    from typing import Any, TypeIs, assert_type
    import inspect

    def isawaitable(x: object) -> TypeIs[Awaitable[Any]]:
        return inspect.isawaitable(x)

    def f(x: Awaitable[int] | int) -> None:
        if isawaitable(x):
            # 型チェッカーはより正確な型 "Awaitable[int] | (int & Awaitable[Any])" を推論することもあります
            assert_type(x, Awaitable[int])
        else:
            assert_type(x, int)

入力型に :term:`assignable` でない型に絞り込むことはエラーです::

    from typing import TypeIs

    def is_str(x: int) -> TypeIs[str]:  # 型チェッカーエラー
        ...

``TypeIs`` はコールバックプロトコルや ``Callable`` :term:`special form` などの戻り値の型としても有効です。 これらのコンテキストでは、bool のサブタイプとして扱われます。 例えば、``Callable[..., TypeIs[int]]`` は ``Callable[..., bool]`` に代入可能です。

``TypeGuard`` とは異なり、``TypeIs`` はその引数型において不変です： ``TypeIs[B]`` は ``TypeIs[A]`` のサブタイプではありません、たとえ ``B`` が ``A`` のサブタイプであってもです。 次の例を考えてみましょう::

    def takes_narrower(x: int | str, narrower: Callable[[object], TypeIs[int]]):
        if narrower(x):
            print(x + 1)  # x は int です
        else:
            print("Hello " + x)  # x は str です

    def is_bool(x: object) -> TypeIs[bool]:
        return isinstance(x, bool)

    takes_narrower(1, is_bool)  # エラー: is_bool は TypeIs[int] ではありません

（``bool`` は ``int`` のサブタイプであることに注意してください。） このコードはランタイムで失敗します。なぜなら、narrower は ``False`` を返し（1 は ``bool`` ではありません）、``takes_narrower()`` の ``else`` ブランチが実行されるためです。 ``takes_narrower(1, is_bool)`` の呼び出しが許可されていた場合、型チェッカーはこのエラーを検出できません。
