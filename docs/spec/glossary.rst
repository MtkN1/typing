.. _`glossary`:

用語集
==========================================================================================

このセクションでは、仕様の他の場所で使用される可能性のあるいくつかの用語を定義します。

.. glossary::

   annotation expression
      アノテーション内で使用することが有効な式。 これは通常、追加の :term:`type qualifiers <type qualifier>` を伴う :term:`type expression` です。 詳細については、 :ref:`"Type and annotation expression" <annotation-expression>` を参照してください。

   assignable
      型 ``B`` が型 ``A`` に「代入可能」である場合、型チェッカーは ``x: A = b`` の代入でエラーを出さないようにする必要があります。ここで、 ``b`` は型が ``B`` である式です。 同様に、関数呼び出しと戻り値についても同様です。 ``f(b)`` では、 ``def f(x: A): ...`` と ``def f(...) -> A: ...`` 内の ``return b`` は、 ``B`` が ``A`` に代入可能である場合にのみ有効です（型エラーではありません）。 この場合、 ``A`` は ``B`` から「代入可能」です。 :term:`fully static types <fully static type>` の場合、「代入可能」は「 :term:`subtype` 」と同義であり、「代入可能」は「 :term:`supertype` 」と同義です。 :term:`gradual types <gradual type>` の場合、型 ``B`` は型 ``A`` に代入可能です。 ``A`` と ``B`` の完全に静的な :term:`materializations <materialize>` ``A'`` と ``B'`` が存在し、 ``B'`` が ``A'`` のサブタイプである場合に限ります。 詳細については、 :ref:`type-system-concepts` を参照してください。

   consistent
      2 つの :term:`fully static types <fully static type>` は、 :term:`equivalent` である場合、「一致」しています。 2 つの漸進的な型は、同じ型に :term:`materialize` できる場合、「一致」しています。 詳細については、 :ref:`type-system-concepts` を参照してください。 2 つの型が一致している場合、それらは両方とも互いに :term:`assignable` です。

   consistent subtype
      「一貫したサブタイプ」は「 :term:`assignable` 」と同義です（「一貫したスーパータイプ」は「代入可能」と同義です）。 詳細については、 :ref:`type-system-concepts` を参照してください。

   distribution
      リリースを公開および配布するために使用されるパッケージ化されたファイル。 (:pep:`426`)

   equivalent
      2 つの :term:`fully static types <fully static type>` ``A`` と ``B`` は、 ``A`` が ``B`` の :term:`subtype` であり、 ``B`` が ``A`` の :term:`subtype` である場合、同等です。 これは、 ``A`` と ``B`` が同じセットの可能なランタイムオブジェクトを表すことを意味します。 2 つの漸進的な型 ``A`` と ``B`` は、 ``A`` のすべての :term:`materializations <materialize>` が ``B`` の materializations でもあり、 ``B`` のすべての materializations が ``A`` の materializations でもある場合、同等です。

   fully static type
      型に :term:`gradual form` が含まれていない場合、その型は「完全に静的」です。 完全に静的な型は、可能なランタイム値のセットを表します。 完全に静的な型は、 :term:`subtype` 関係に参加します。 詳細については、 :ref:`type-system-concepts` を参照してください。

   gradual form
      漸進的な形式は、 :term:`type expression` の要素であり、それが一部である型を :term:`fully static type` ではなく、可能な静的型のセットの表現にします。 詳細については、 :ref:`type-system-concepts` を参照してください。 主な漸進的な形式は :ref:`Any` です。 省略記号（ ``...`` ）は、いくつかのコンテキストでは漸進的な形式です。 :ref:`Callable` 型で使用される場合、および ``tuple[Any, ...]`` で使用される場合（ただし、他の :ref:`tuple <tuples>` 型ではない場合）に漸進的な形式です。

   gradual type
      Python 型システムのすべての型は「漸進的」です。 漸進的な型は、 :term:`fully static type` 、または :ref:`Any` 、または ``Any`` または他の :term:`gradual form` を含む型である場合があります。 漸進的な型は、必ずしも単一の可能なランタイム値のセットを表すわけではありません。 代わりに、可能な静的型のセット（可能なランタイム値のセットのセット）を表すことができます。 漸進的な型は、 :term:`subtype` 関係には参加しませんが、 :term:`consistency <consistent>` および :term:`assignability <assignable>` には参加します。 それらは、より静的な、または完全に静的な型に :term:`materialized <materialize>` することができます。 詳細については、 :ref:`type-system-concepts` を参照してください。

   inline
      インライン型アノテーションは、 :pep:`526` および :pep:`3107` 構文を使用してランタイムコードに含まれるアノテーションです（ファイル名は ``.py`` で終わります）。

   materialize
      :term:`gradual type` は、 :ref:`Any` を他の任意の型に置き換えるか、 :ref:`Callable` 型の `...` を型のリストに置き換えるか、 ``tuple[Any, ...]`` を特定の長さのタプル型に置き換えることにより、より静的な型（おそらく :term:`fully static type` ）に materialized できます。 この materialization 関係は、漸進的な型の :term:`assignability <assignable>` を定義するための鍵です。 詳細については、 :ref:`type-system-concepts` を参照してください。

   module
      Python ランタイムコードまたはスタブ型情報を含むファイル。

   narrow
      :term:`fully static type` ``B`` は、 ``B`` が ``A`` の :term:`subtype` であり、 ``B`` が ``A`` と :term:`equivalent` ではない場合、完全に静的な型 ``A`` よりも狭いです。 これは、 ``B`` が ``A`` によって表される可能なオブジェクトの適切なサブセットを表すことを意味します。 「型の絞り込み」とは、型チェッカーが、代入またはランタイムチェックの値のために、制御フローの一部の場所で名前または式がより狭い型を持たなければならないと推測することです。

   nominal
      名目型（例：クラス名）は、その型の ``__class__`` またはそのサブクラスのいずれかのサブクラスの値のセットを表します。 対照的に、 :term:`structural` 型を参照してください。

   package
      Python モジュールに名前空間を提供するディレクトリまたはディレクトリ。 （パッケージと :term:`distributions <distribution>` の違いに注意してください。ほとんどのディストリビューションはインストールする 1 つのパッケージにちなんで名付けられていますが、一部のディストリビューションは複数のパッケージをインストールします。）

   special form
      特殊形式は、型システム内で特別な意味を持つオブジェクトであり、言語文法のキーワードに匹敵します。 例としては、 ``Any`` 、 ``Generic`` 、 ``Literal`` 、および ``TypedDict`` があります。 特殊形式は、 :ref:`type expressions <type-expression>` 内で使用できる場合がありますが、常にではありません。 特殊形式は通常、 :py:mod:`typing` モジュールまたは同等の ``typing_extensions`` からインポートできますが、一部の特殊形式は他のモジュールに配置されています。

   structural
      構造型（例： :ref:`Protocols` 、 :ref:`TypedDict` ）は、 ``__class__`` ではなく、プロパティ（例：属性、メソッド、辞書のキー/値の型）によって値のセットを定義します。 :ref:`Callable` 型も構造的です。 呼び出し可能な型は、サブクラス関係ではなく、シグネチャに基づいて他の呼び出し可能な型のサブタイプです。 対照的に、 :term:`nominal` 型を参照してください。

   stub
      ランタイムコードが含まれていない型情報のみを含むファイル（ファイル名は ``.pyi`` で終わります）。 詳細については、 :ref:`stub-files` を参照してください。

   subtype
      :term:`fully static type` ``B`` は、 ``B`` によって表される可能なランタイム値のセットが ``A`` によって表される可能なランタイム値のセットのサブセットである場合にのみ、完全に静的な型 ``A`` のサブタイプです。 :term:`nominal` 型（クラス）の場合、サブタイプは継承によって定義されます。 :term:`structural` 型の場合、サブタイプは共有される属性/メソッドまたはキーのセットによって定義されます。 サブタイプは :term:`supertype` の逆です。 完全に静的ではない型は、他の型のサブタイプまたはスーパータイプではありませんが、 :term:`materialization <materialize>` を介して他の型に :term:`assignable` できます。 詳細については、 :ref:`type-system-concepts` を参照してください。

   supertype
      :term:`fully static type` ``A`` は、 ``A`` によって表される可能なランタイム値のセットが ``B`` によって表される可能なランタイム値のセットのスーパーセットである場合にのみ、完全に静的な型 ``B`` のスーパータイプです。 スーパータイプは :term:`subtype` の逆です。 詳細については、 :ref:`type-system-concepts` を参照してください。

   type expression
      型を表す式。 型システムは、 :term:`annotation expression` 内および他のいくつかのコンテキストで型式の使用を要求します。 詳細については、 :ref:`"Type and annotation expression" <type-expression>` を参照してください。

   type qualifier
      型修飾子は、 :term:`special form` であり、 :term:`type expression` を修飾して :term:`annotation expression` を形成します。 たとえば、型修飾子 :ref:`Final <uppercase-final>` は、注釈付きの値がオーバーライドまたは変更されないことを示すために型の周りに使用できます。 この用語は、 :ref:`@final <at-final>` デコレータなど、異なる構文コンテキストを使用して型を変更する他の特殊形式にも使用されます。

   wide
      :term:`fully static type` ``A`` は、 ``B`` が ``A`` の :term:`subtype` であり、 ``B`` が ``A`` と :term:`equivalent` ではない場合にのみ、完全に静的な型 ``B`` よりも広いです。 これは、 ``A`` が ``B`` によって表される可能な値の適切なスーパーセットを表すことを意味します。 また、 ":term:`narrow`" も参照してください。
