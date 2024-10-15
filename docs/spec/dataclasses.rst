.. _`dataclasses`:

データクラス
==========================================================================================

型チェッカーは、:py:mod:`dataclasses` モジュールを通じて作成されたデータクラスをサポートする必要があります。 さらに、型システムには、サードパーティのクラスを標準のデータクラスのように動作させるメカニズムが含まれています。

.. _`dataclass-transform`:

``dataclass_transform`` デコレーター
------------------------------------------------------------------------------------------

（元々 :pep:`681` で指定されています。）

仕様
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

この仕様では、:py:mod:`typing` モジュールに :py:func:`~typing.dataclass_transform` という名前のデコレーター関数について説明します。 このデコレーターは、デコレーター自体である関数、クラス、またはメタクラスに適用できます。 ``dataclass_transform`` の存在は、デコレーター関数、クラス、またはメタクラスがクラスを変換し、データクラスのような動作を付与するランタイムの「マジック」を実行することを静的型チェッカーに伝えます。

``dataclass_transform`` が関数に適用される場合、デコレーターとして装飾された関数を使用することは、データクラスのようなセマンティクスを適用するものと見なされます。 関数にオーバーロードがある場合、``dataclass_transform`` デコレーターは関数の実装またはオーバーロードのいずれか 1 つに適用できますが、複数のオーバーロードには適用できません。 オーバーロードに適用される場合でも、``dataclass_transform`` デコレーターは関数のすべての使用に影響を与えます。

``dataclass_transform`` がクラスに適用される場合、データクラスのようなセマンティクスは、装飾されたクラスから直接または間接的に派生するクラスや、装飾されたクラスをメタクラスとして使用するクラスに対して適用されるものと見なされます。 装飾されたクラスおよびその基本クラスの属性はフィールドとは見なされません。

各アプローチの例は、以下のセクションに示されています。 各例では、``CustomerModel`` クラスをデータクラスのようなセマンティクスで作成します。 装飾されたオブジェクトの実装は簡潔さのために省略されていますが、クラスを次のように変更するものと仮定します。

* クラス内およびその親クラス内で宣言されたデータフィールドを使用して ``__init__`` メソッドを合成します。
* ``__eq__`` および ``__ne__`` メソッドを合成します。

型チェッカーは、``CustomerModel`` クラスが合成された ``__init__`` メソッドを使用してインスタンス化できることを認識します。

.. code-block:: python

  # 位置引数を使用
  c1 = CustomerModel(327, "John Smith")

  # キーワード引数を使用
  c2 = CustomerModel(id=327, name="John Smith")

  # これらの呼び出しはランタイムエラーを生成し、静的型チェッカーによってエラーとしてフラグ付けされるべきです。
  c3 = CustomerModel()
  c4 = CustomerModel(327, first_name="John")
  c5 = CustomerModel(327, "John Smith", 0)

デコレーター関数の例
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

.. code-block:: python

  _T = TypeVar("_T")

  # ``create_model`` デコレーターはライブラリによって定義されます。
  # これはタイプスタブまたはインラインで行うことができます。
  @typing.dataclass_transform()
  def create_model(cls: Type[_T]) -> Type[_T]:
      cls.__init__ = ...
      cls.__eq__ = ...
      cls.__ne__ = ...
      return cls

  # ``create_model`` デコレーターは次のように新しいモデルクラスを作成するために使用できます。
  @create_model
  class CustomerModel:
      id: int
      name: str

クラスの例
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

.. code-block:: python

  # ``ModelBase`` クラスはライブラリによって定義されます。 これはタイプスタブまたはインラインで行うことができます。
  @typing.dataclass_transform()
  class ModelBase: ...

  # ``ModelBase`` クラスは次のように新しいモデルサブクラスを作成するために使用できます。
  class CustomerModel(ModelBase):
      id: int
      name: str

メタクラスの例
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

.. code-block:: python

  # ``ModelMeta`` メタクラスおよび ``ModelBase`` クラスはライブラリによって定義されます。 これはタイプスタブまたはインラインで行うことができます。
  @typing.dataclass_transform()
  class ModelMeta(type): ...

  class ModelBase(metaclass=ModelMeta): ...

  # ``ModelBase`` クラスは次のように新しいモデルサブクラスを作成するために使用できます。
  class CustomerModel(ModelBase):
      id: int
      name: str

デコレーター関数およびクラス/メタクラスのパラメータ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

データクラスのような機能を提供するデコレーター関数、クラス、またはメタクラスは、特定の動作を変更するパラメータを受け入れる場合があります。 この仕様では、データクラス変換によって使用される場合に静的型チェッカーが尊重しなければならない次のパラメータを定義します。 これらのパラメータは bool 引数を受け入れ、bool 値（``True`` または ``False``）を静的に評価できる必要があります。

* ``eq``、``order``、``frozen``、``init`` および ``unsafe_hash`` は、標準ライブラリのデータクラスでサポートされているパラメータであり、その意味は :pep:`PEP 557 <557#id7>` で定義されています。
* ``kw_only``、``match_args`` および ``slots`` は、標準ライブラリのデータクラスでサポートされているパラメータであり、Python 3.10 で初めて導入されました。

``dataclass_transform`` パラメータ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``dataclass_transform`` のパラメータは、デフォルトの動作を基本的にカスタマイズするためのものです。

.. code-block:: python

  _T = TypeVar("_T")

  def dataclass_transform(
      *,
      eq_default: bool = True,
      order_default: bool = False,
      kw_only_default: bool = False,
      frozen_default: bool = False,
      field_specifiers: tuple[type | Callable[..., Any], ...] = (),
      **kwargs: Any,
  ) -> Callable[[_T], _T]: ...

* ``eq_default`` は、呼び出し元が ``eq`` パラメータを省略した場合に ``True`` または ``False`` と見なすかどうかを示します。 指定されていない場合、``eq_default`` は True にデフォルト設定されます（データクラスのデフォルトの仮定）。
* ``order_default`` は、呼び出し元が ``order`` パラメータを省略した場合に ``True`` または ``False`` と見なすかどうかを示します。 指定されていない場合、``order_default`` は False にデフォルト設定されます（データクラスのデフォルトの仮定）。
* ``kw_only_default`` は、呼び出し元が ``kw_only`` パラメータを省略した場合に ``True`` または ``False`` と見なすかどうかを示します。 指定されていない場合、``kw_only_default`` は False にデフォルト設定されます（データクラスのデフォルトの仮定）。
* ``frozen_default`` は、呼び出し元が ``frozen`` パラメータを省略した場合に ``True`` または ``False`` と見なすかどうかを示します。 指定されていない場合、``frozen_default`` は False にデフォルト設定されます（データクラスのデフォルトの仮定）。
* ``field_specifiers`` は、フィールドを記述するサポートされるクラスの静的リストを指定します。 一部のライブラリは、フィールド指定子のインスタンスを割り当てる関数も提供しており、これらの関数もこのタプルに指定できます。 指定されていない場合、``field_specifiers`` は空のタプル（フィールド指定子はサポートされていない）にデフォルト設定されます。 標準のデータクラスの動作は、``Field`` と呼ばれる 1 種類のフィールド指定子と、このクラスのインスタンスを作成するヘルパー関数（``field``）のみをサポートするため、標準ライブラリのデータクラスの動作を説明する場合、タプル引数 ``(dataclasses.Field, dataclasses.field)`` を提供します。
* ``kwargs`` は、``dataclass_transform`` に任意の追加のキーワード引数を渡すことを可能にします。 これにより、型チェッカーは ``typing.py`` の変更を待たずに実験的なパラメータをサポートする自由を得ることができます。 型チェッカーは、認識されないパラメータに対してエラーを報告する必要があります。

将来的には、ユーザーコードで一般的な動作をサポートするために必要に応じて、``dataclass_transform`` に追加のパラメータを追加する場合があります。 これらの追加は、追加の PEP を介さずに typing-sig での合意に達した後に行われます。

次のセクションでは、これらのパラメータの使用方法を示す追加の例を提供します。

デコレーター関数の例
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

.. code-block:: python

  # ``create_model`` 関数がキーワード専用パラメータを合成された ``__init__`` メソッドに仮定することを示します。 ``kw_only=False`` で呼び出されない限り。 常に順序関連のメソッドを合成し、この動作を上書きする方法を提供しません。
  @typing.dataclass_transform(kw_only_default=True, order_default=True)
  def create_model(
      *,
      frozen: bool = False,
      kw_only: bool = True,
  ) -> Callable[[Type[_T]], Type[_T]]: ...

  # このデコレーターがこのライブラリからインポートされたコードによってどのように使用されるかの例：
  @create_model(frozen=True, kw_only=False)
  class CustomerModel:
      id: int
      name: str

クラスの例
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

.. code-block:: python

  # このクラスから派生するクラスが比較メソッドを合成することをデフォルトとすることを示します。
  @typing.dataclass_transform(eq_default=True, order_default=True)
  class ModelBase:
      def __init_subclass__(
          cls,
          *,
          init: bool = True,
          frozen: bool = False,
          eq: bool = True,
          order: bool = True,
      ):
          ...

  # このクラスがこのライブラリからインポートされたコードによってどのように使用されるかの例：
  class CustomerModel(
      ModelBase,
      init=False,
      frozen=True,
      eq=False,
      order=False,
  ):
      id: int
      name: str

メタクラスの例
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

.. code-block:: python

  # このメタクラスを使用するクラスが比較メソッドを合成することをデフォルトとすることを示します。
  @typing.dataclass_transform(eq_default=True, order_default=True)
  class ModelMeta(type):
      def __new__(
          cls,
          name,
          bases,
          namespace,
          *,
          init: bool = True,
          frozen: bool = False,
          eq: bool = True,
          order: bool = True,
      ):
          ...

  class ModelBase(metaclass=ModelMeta):
      ...

  # このクラスがこのライブラリからインポートされたコードによってどのように使用されるかの例：
  class CustomerModel(
      ModelBase,
      init=False,
      frozen=True,
      eq=False,
      order=False,
  ):
      id: int
      name: str

フィールド指定子
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

データクラスのようなセマンティクスをサポートするほとんどのライブラリは、クラス定義がクラス内の各フィールドに関する追加のメタデータを提供できるようにする 1 つ以上の「フィールド指定子」タイプを提供します。 このメタデータは、たとえばデフォルト値を記述したり、フィールドが合成された ``__init__`` メソッドに含まれるかどうかを示したりすることができます。

追加のメタデータが必要ない場合、フィールド指定子を省略できます。

.. code-block:: python

  @dataclass
  class Employee:
      # フィールド指定子なしのフィールド
      name: str

      # フィールド指定子クラスインスタンスを使用するフィールド
      age: Optional[int] = field(default=None, init=False)

      # デフォルト値を記述するための型注釈と単純な初期化子を持つフィールド
      is_paid_hourly: bool = True

      # 型注釈が提供されていないため、フィールドではなく（クラス変数）
      office_number = "unassigned"

フィールド指定子のパラメータ
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

データクラスのようなセマンティクスをサポートし、フィールド指定子クラスをサポートするライブラリは、通常、これらのフィールド指定子を構築するために共通のパラメータ名を使用します。 この仕様では、静的型チェッカーが理解しなければならないパラメータの名前と意味を正式化します。 これらの標準化されたパラメータはキーワード専用でなければなりません。

これらのパラメータは、``compare`` や ``hash`` など、型チェックに影響を与えないものを除く、:py:func:`dataclasses.field` でサポートされているもののスーパーセットです。

フィールド指定子クラスは、コンストラクタ内で他のパラメータを使用することができ、これらのパラメータは位置指定であり、他の名前を使用することができます。

* ``init`` は、フィールドが合成された ``__init__`` メソッドに含まれるかどうかを示すオプションの bool パラメータです。 指定されていない場合、``init`` は True にデフォルト設定されます。 フィールド指定子関数は、リテラル bool 値型（``Literal[False]`` または ``Literal[True]``）を使用して ``init`` の値を暗黙的に指定するオーバーロードを使用できます。
* ``default`` は、フィールドのデフォルト値を提供するオプションのパラメータです。
* ``default_factory`` は、フィールドのデフォルト値を返すランタイムコールバックを提供するオプションのパラメータです。 ``default`` または ``default_factory`` のいずれも指定されていない場合、フィールドにはデフォルト値がないと見なされ、クラスのインスタンス化時に値を提供する必要があります。
* ``factory`` は ``default_factory`` のエイリアスです。 標準ライブラリのデータクラスは ``default_factory`` という名前を使用しますが、attrs は多くのシナリオで ``factory`` という名前を使用するため、このエイリアスは attrs をサポートするために必要です。
* ``kw_only`` は、フィールドがキーワード専用としてマークされるかどうかを示すオプションの bool パラメータです。 True の場合、フィールドはキーワード専用になります。 False の場合、キーワード専用にはなりません。 指定されていない場合、``dataclass_transform`` で装飾されたオブジェクトの ``kw_only`` パラメータの値が使用されます。 それも指定されていない場合、``dataclass_transform`` の ``kw_only_default`` の値が使用されます。
* ``alias`` は、フィールドの代替名を提供するオプションの str パラメータです。 この代替名は、合成された ``__init__`` メソッドで使用されます。
* ``converter`` は、フィールドに値を割り当てる際に使用される呼び出し可能なオプションのパラメータです。

``default``、``default_factory`` および ``factory`` のいずれかを複数指定することはエラーです。

次の例は上記を示しています。

.. code-block:: python

  # ライブラリコード（タイプスタブまたはインライン内）
  # このライブラリでは、リゾルバを渡すと init は False でなければならず、Literal[False] を持つオーバーロードがそれを強制します。
  @overload
  def model_field(
          *,
          default: Optional[Any] = ...,
          resolver: Callable[[], Any],
          init: Literal[False] = False,
      ) -> Any: ...

  @overload
  def model_field(
          *,
          default: Optional[Any] = ...,
          resolver: None = None,
          init: bool = True,
      ) -> Any: ...

  @typing.dataclass_transform(
      kw_only_default=True,
      field_specifiers=(model_field, ))
  def create_model(
      *,
      init: bool = True,
  ) -> Callable[[Type[_T]], Type[_T]]: ...

  # このライブラリからインポートされたコード：
  @create_model(init=False)
  class CustomerModel:
      id: int = model_field(resolver=lambda : 0)
      name: str

ランタイムの動作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ランタイムでは、``dataclass_transform`` デコレーターの唯一の効果は、装飾された関数またはクラスに ``__dataclass_transform__`` という名前の属性を設定してインスペクションをサポートすることです。 属性の値は、``dataclass_transform`` パラメータの名前をその値にマッピングする辞書である必要があります。

たとえば：

.. code-block:: python

  {
    "eq_default": True,
    "order_default": False,
    "kw_only_default": False,
    "field_specifiers": (),
    "kwargs": {}
  }

データクラスのセマンティクス
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

明示的に述べられていない限り、``dataclass_transform`` で影響を受けるクラスは、``dataclass_transform`` で装飾されたクラスを継承するか、``dataclass_transform`` で装飾された関数で装飾されることによって、標準ライブラリの :func:`~dataclasses.dataclass` のように動作するものと見なされます。

これには、次のセマンティクスが含まれますが、これに限定されません。

* 凍結されたデータクラスは、非凍結データクラスを継承できません。 ``dataclass_transform`` で装飾されたクラスは、凍結されているとも非凍結されているとも見なされないため、凍結されたクラスがそれを継承することができます。 同様に、``dataclass_transform`` で装飾されたメタクラスを直接指定するクラスは、凍結されているとも非凍結されているとも見なされません。

  次のクラスの例を考えてみましょう。

  .. code-block:: python

    # ModelBase は ``dataclass_transform`` で装飾されているため、「凍結」または「非凍結」と見なされません。
    @typing.dataclass_transform()
    class ModelBase(): ...

    # Vehicle は「frozen=True」を指定していないため、非凍結と見なされます。
    class Vehicle(ModelBase):
        name: str

    # Car は凍結されたクラスであり、非凍結クラスである Vehicle から派生しています。 これはエラーです。
    class Car(Vehicle, frozen=True):
        wheel_count: int

  そして、これらの類似したメタクラスの例：

  .. code-block:: python

    @typing.dataclass_transform()
    class ModelMeta(type): ...

    # ModelBase は、ModelMeta をメタクラスとして直接指定しているため、「凍結」または「非凍結」と見なされません。
    class ModelBase(metaclass=ModelMeta): ...

    # Vehicle は「frozen=True」を指定していないため、非凍結と見なされます。
    class Vehicle(ModelBase):
        name: str

    # Car は凍結されたクラスであり、非凍結クラスである Vehicle から派生しています。 これはエラーです。
    class Car(Vehicle, frozen=True):
        wheel_count: int

* フィールドの順序と継承は、`Python ドキュメント <https://docs.python.org/3/library/dataclasses.html#inheritance>`__ で指定されたルールに従うものと見なされます。 これには、オーバーライドの影響（親クラスですでに定義されているフィールドを子クラスで再定義すること）が含まれます。

* :pep:`PEP 557 <557#post-init-parameters>` は、デフォルト値のないすべてのフィールドがデフォルト値のあるフィールドの前に表示される必要があることを示しています。 PEP 557 では明示的に述べられていませんが、このルールは ``init=False`` の場合には無視され、この仕様でもその状況ではこの要件を無視します。 同様に、``__init__`` のキーワード専用パラメータが使用される場合、この順序を強制する必要はないため、``kw_only`` セマンティクスが有効な場合、このルールは強制されません。

* ``dataclass`` と同様に、クラス内で明示的に宣言されたメソッドがある場合、メソッドの合成はスキップされます。 基本クラスのメソッド宣言は、メソッドの合成をスキップする原因にはなりません。

  たとえば、クラスが ``__init__`` メソッドを明示的に宣言している場合、そのクラスには ``__init__`` メソッドは合成されません。

* KW_ONLY センチネル値は、`Python ドキュメント <https://docs.python.org/3/library/dataclasses.html#dataclasses.KW_ONLY>`_ および `bpo-43532 <https://bugs.python.org/issue43532>`_ で説明されているようにサポートされています。

* ClassVar 属性はデータクラスフィールドとは見なされず、`データクラスメカニズムによって無視されます <https://docs.python.org/3/library/dataclasses.html#class-variables>`_。

* データクラスフィールドは ``Final[...]`` で注釈を付けることができます。 たとえば、データクラス本体内の ``x: Final[int]`` は、合成された ``__init__`` メソッドで初期化され、その後は代入できないデータクラスフィールド ``x`` を指定します。 クラス本体内で初期化された ``Final`` データクラスフィールドは、明示的に ``ClassVar`` で注釈されない限り、クラス属性ではありません。 たとえば、``x: Final[int] = 3`` は、合成された ``__init__`` メソッドでデフォルト値 ``3`` を持つデータクラスフィールド ``x`` です。 データクラスの最終クラス変数は、たとえば ``x: ClassVar[Final[int]] = 3`` のように明示的に注釈を付ける必要があります。

コンバーター
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``converter`` パラメータは、関連する属性に値を割り当てる際に使用される呼び出し可能なものを提供するためにフィールド定義で指定できます。 この機能により、属性の割り当て中に自動的な型変換と検証が可能になります。

コンバーターの動作：

* コンバーターは、デフォルト値の割り当て、合成された ``__init__`` メソッドでの割り当て、および直接の属性設定（例：``obj.attr = value``）を含むすべての属性割り当てに使用されます。
* コンバーターは属性の読み取り時には使用されません。属性はすでに変換されているはずです。

コンバーターの型付けルール：

* ``converter`` は、単一の位置引数を受け入れる呼び出し可能なものでなければなりません（ただし、他のオプションの引数を受け入れることができますが、型付け目的では無視されます）。
* 最初の位置引数の型は、フィールドの合成された ``__init__`` パラメータの型を提供します。
* 呼び出し可能なものの戻り値の型は、フィールドの宣言された型に代入可能でなければなりません。
* ``default`` または ``default_factory`` が提供されている場合、デフォルト値の型は ``converter`` の最初の位置引数に代入可能でなければなりません。

使用例：

.. code-block:: python

    def str_or_none(x: Any) -> str | None:
        return str(x) if x is not None else None

    @custom_dataclass
    class Example:
        int_field: int = custom_field(converter=int)
        str_field: str | None = custom_field(converter=str_or_none)
        path_field: pathlib.Path = custom_field(
            converter=pathlib.Path,
            default="default/path.txt"
        )

    # 使用例
    example = Example("123", None, "some/path")
    # example.int_field == 123
    # example.str_field == None
    # example.path_field == pathlib.Path("some/path")

未定義の動作
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

複数の ``dataclass_transform`` デコレーターが見つかった場合、単一の関数（そのオーバーロードを含む）、単一のクラス、またはクラス階層内で、結果の動作は未定義です。 ライブラリの作成者はこれらのシナリオを避けるべきです。
