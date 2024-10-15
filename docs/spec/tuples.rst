.. _`tuples`:

タプル
==========================================================================================

``tuple`` クラスには、他のクラスとは異なる特別な動作と特性があります。 最も明らかな違いは、``tuple`` が可変長であることです。 つまり、任意の数の型引数をサポートします。 実行時には、タプル内に含まれるオブジェクトのシーケンスは構築時に固定されます。 構築後に要素を追加、削除、並べ替え、または置換することはできません。 これらの特性は、以下に説明するように、サブタイプルールやその他の動作に影響を与えます。


タプル型の形式
------------------------------------------------------------------------------------------

タプルの型は、要素の型を列挙することで表現できます。 例えば、``tuple[int, int, str]`` は、``int``、もう一つの ``int``、および ``str`` を含むタプルです。

空のタプルは ``tuple[()]`` として注釈を付けることができます。

任意の長さの同種のタプルは、1 つの型と省略記号を使用して表現できます。 例えば、``tuple[int, ...]``。 この型は、0 個以上の ``int`` 要素を含むタプルの共用体と同等です（``tuple[()] | tuple[int] | tuple[int, int] | tuple[int, int, int] | ...``）。 任意の長さの同種のタプルは、時々「無制限タプル」と呼ばれます。 これらの用語はどちらも型指定の仕様内で使用され、同じ概念を指します。

型 ``tuple[Any, ...]`` は特別で、すべてのタプル型と :term:`consistent` であり、任意の長さのタプルに :term:`assignable` です。 これは、段階的な型付けに役立ちます。 型 ``tuple``（型引数が提供されていない場合）は、``tuple[Any, ...]`` と同等です。

任意の長さのタプルには、正確に 2 つの型引数（型と省略記号）があります。 省略記号を使用する他のタプル形式は無効です::

    t1: tuple[int, ...]  # OK
    t2: tuple[int, int, ...]  # 無効
    t3: tuple[...]  # 無効
    t4: tuple[..., int]  # 無効
    t5: tuple[int, ..., int]  # 無効
    t6: tuple[*tuple[str], ...]  # 無効
    t7: tuple[*tuple[str, ...], ...]  # 無効


アンパックされたタプル形式
------------------------------------------------------------------------------------------

タプルのアンパック形式（アンパック演算子 ``*`` を使用）をタプル型引数リスト内で使用できます。 例えば、``tuple[int, *tuple[str]]`` は ``tuple[int, str]`` と同等です。 無制限タプルのアンパックは、そのまま無制限タプルとして保持されます。 つまり、``*tuple[int, ...]`` は ``*tuple[int, ...]`` のままです。 より簡単な形式はありません。 これにより、``tuple[int, *tuple[str, ...], str]`` のような型を指定できます。最初の要素が ``int`` 型であることが保証され、最後の要素が ``str`` 型であることが保証され、中間の要素が 0 個以上の ``str`` 型の要素であるタプル型です。 型 ``tuple[*tuple[int, ...]]`` は ``tuple[int, ...]`` と同等です。

アンパックされた ``*tuple[Any, ...]`` が別のタプル内に埋め込まれている場合、そのタプルの部分は任意の長さのタプルと :term:`consistent` です。

別のタプル内で使用できる無制限タプルは 1 つだけです::

    t1: tuple[*tuple[str], *tuple[str]]  # OK
    t2: tuple[*tuple[str, *tuple[str, ...]]]  # OK
    t3: tuple[*tuple[str, ...], *tuple[int, ...]]  # 型エラー
    t4: tuple[*tuple[str, *tuple[str, ...]], *tuple[int, ...]]  # 型エラー

アンパックされた TypeVarTuple は、このルールの文脈では無制限タプルとしてカウントされます::

    def func[*Ts](t: tuple[*Ts]):
        t5: tuple[*tuple[str], *Ts]  # OK
        t6: tuple[*tuple[str, ...], *Ts]  # 型エラー

``*`` 構文は Python 3.11 以降が必要です。 古いバージョンの Python では、``typing.Unpack`` :term:`special form` を使用できます：``tuple[int, Unpack[tuple[str, ...]], int]``。

アンパックされたタプルは、関数シグネチャの ``*args`` パラメータにも使用できます：``def f(*args: *tuple[int, str]): ...``。 アンパックされたタプルは、``TypeVarTuple`` を使用してパラメータ化されたジェネリッククラスや型変数を特殊化するためにも使用できます。 詳細については、:ref:`args_as_typevartuple` を参照してください。


型の互換性ルール
------------------------------------------------------------------------------------------

タプルの内容は不変であるため、タプルの要素型は共変です。 例えば、``tuple[int, int]`` は ``tuple[float, complex]`` のサブタイプです。

前述のように、任意の長さの同種のタプルは、異なる長さのタプルの共用体と同等です。 つまり、``tuple[()]``、``tuple[int]``、および ``tuple[int, *tuple[int, ...]]`` はすべて ``tuple[int, ...]`` のサブタイプです。 逆は真ではありません。 ``tuple[int, ...]`` は ``tuple[int]`` のサブタイプではありません。

型 ``tuple[Any, ...]`` は任意のタプルと :term:`consistent` です::

    def func(t1: tuple[int], t2: tuple[int, ...], t3: tuple[Any, ...]):
        v1: tuple[int, ...] = t1  # OK
        v2: tuple[Any, ...] = t1  # OK

        v3: tuple[int] = t2  # 型エラー
        v4: tuple[Any, ...] = t2  # OK

        v5: tuple[float, float] = t3  # OK
        v6: tuple[int, *tuple[str, ...]] = t3  # OK


タプルの長さは実行時に不変であるため、型チェッカーは長さチェックを使用してタプルの型を絞り込むことが安全です::

    def func(val: tuple[int] | tuple[str, str] | tuple[int, *tuple[str, ...], int]):
        if len(val) == 1:
            # 型を tuple[int] に絞り込むことができます。
            reveal_type(val)  # tuple[int]

        if len(val) == 2:
            # 型を tuple[str, str] | tuple[int, int] に絞り込むことができます。
            reveal_type(val)  # tuple[str, str] | tuple[int, int]

        if len(val) == 3:
            # 型を tuple[int, str, int] に絞り込むことができます。
            reveal_type(val)  # tuple[int, str, int]

この特性は、シーケンスパターンを使用する ``match`` ステートメント内でタプル型を安全に絞り込むためにも使用できます。

タプル要素が共用体型である場合、タプルを共用体に安全に展開できます。 例えば、``tuple[int | str]`` は ``tuple[int]`` | ``tuple[str]`` と同等です。 複数の要素が共用体型である場合、完全な展開はすべての組み合わせを考慮する必要があります。 例えば、``tuple[int | str, int | str]`` は ``tuple[int, int] | tuple[int, str] | tuple[str, int] | tuple[str, str]`` と同等です。 無制限タプルはこの方法で展開することはできません。

型チェッカーは、タプル型を絞り込む際にこの等価性ルールを安全に使用できます::

    def func(subj: tuple[int | str, int | str]):
        match subj:
            case x, str():
                reveal_type(subj)  # tuple[int | str, str]
            case y:
                reveal_type(subj)  # tuple[int | str, int]

``tuple`` クラスは ``Sequence[T_co]`` から派生しており、``T_co`` は共変（非可変長）型変数です。 ``T_co`` の特殊化された型は、すべての要素型のスーパータイプとして型チェッカーによって計算されるべきです。 例えば、``tuple[int, *tuple[str, ...]]`` は ``Sequence[int | str]`` または ``Sequence[object]`` のサブタイプです。

長さ 0 のタプル（``tuple[()]``）は ``Sequence[Never]`` のサブタイプです。
