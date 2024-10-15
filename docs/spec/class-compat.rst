.. _`class-compat`:

クラス型の代入可能性
==========================================================================================

.. _`classvar`:

``ClassVar``
------------------------------------------------------------------------------------------

（元々 :pep:`526` で指定されています。）

:term:`type qualifier` ``ClassVar[T]`` は :py:mod:`typing` モジュールに存在します。 これは単一の引数のみを受け入れる必要があり、有効な型である必要があります。 クラスインスタンスに設定されるべきではないクラス変数を注釈するために使用されます。 この制限は静的チェッカーによって強制されますが、実行時には強制されません。

型注釈は、クラス本体およびメソッド内のクラス変数およびインスタンス変数を注釈するために使用できます。 特に、値のない表記 ``a: int`` は、``__init__`` または ``__new__`` で初期化されるべきインスタンス変数を注釈することを可能にします。 構文は次のとおりです::

  class BasicStarship:
      captain: str = 'Picard'               # デフォルト値を持つインスタンス変数
      damage: int                           # デフォルト値を持たないインスタンス変数
      stats: ClassVar[dict[str, int]] = {}  # クラス変数

ここで ``ClassVar`` は :py:mod:`typing` モジュールによって定義された :term:`special form` であり、この変数がインスタンスに設定されるべきではないことを静的型チェッカーに示します。

``ClassVar`` パラメータには、ネストのレベルに関係なく、型変数を含めることはできません。 ``ClassVar[T]`` および ``ClassVar[list[set[T]]]`` は、``T`` が型変数である場合、どちらも無効です。

これは、より詳細な例で説明できます。 このクラスでは::

  class Starship:
      captain = 'Picard'
      stats = {}

      def __init__(self, damage, captain=None):
          self.damage = damage
          if captain:
              self.captain = captain  # それ以外の場合はデフォルトを保持

      def hit(self):
          Starship.stats['hits'] = Starship.stats.get('hits', 0) + 1

``stats`` はクラス変数（多くの異なるゲームごとの統計を追跡するため）であり、``captain`` はクラスで設定されたデフォルト値を持つインスタンス変数です。 この違いは型チェッカーによって認識されないかもしれません。 両方ともクラスレベルで初期化されますが、``captain`` はインスタンス変数の便利なデフォルト値として機能し、``stats`` は真にクラス変数です。 これはすべてのインスタンスで共有されることを意図しています。

両方の変数がクラスレベルで初期化されるため、クラス変数を ``ClassVar[...]`` でラップされた型で注釈することで区別することが有用です。 このようにして、型チェッカーはインスタンス上の同じ名前の属性への偶発的な代入をフラグ付けすることができます。

たとえば、注釈付きのクラスを示します::

  class Starship:
      captain: str = 'Picard'
      damage: int
      stats: ClassVar[dict[str, int]] = {}

      def __init__(self, damage: int, captain: str = None):
          self.damage = damage
          if captain:
              self.captain = captain  # それ以外の場合はデフォルトを保持

      def hit(self):
          Starship.stats['hits'] = Starship.stats.get('hits', 0) + 1

  enterprise_d = Starship(3000)
  enterprise_d.stats = {} # 型チェッカーによってエラーとしてフラグ付けされます
  Starship.stats = {} # これは OK です

便宜上（および慣例として）、インスタンス変数はクラスではなく ``__init__`` または他のメソッドで注釈することができます::

  from typing import Generic, TypeVar
  T = TypeVar('T')

  class Box(Generic[T]):
      def __init__(self, content):
          self.content: T = content

.. _`override`:

``@override``
------------------------------------------------------------------------------------------

（元々 :pep:`698` で指定されています。）

型チェッカーが ``@typing.override`` で装飾されたメソッドに遭遇した場合、そのメソッドが祖先クラスのメソッドまたは属性をオーバーライドしており、オーバーライドされたメソッドの型に :term:`assignable` でない限り、それを型エラーとして扱う必要があります。

.. code-block:: python

    from typing import override

    class Parent:
        def foo(self) -> int:
            return 1

        def bar(self, x: str) -> str:
            return x

    class Child(Parent):
        @override
        def foo(self) -> int:
            return 2

        @override
        def baz(self) -> int:  # 型チェックエラー: 祖先に一致するシグネチャがありません
            return 1

``@override`` デコレーターは、型チェッカーがメソッドを有効なオーバーライドと見なす場所であればどこでも許可されるべきです。 これには、通常のメソッドだけでなく、``@property``、``@staticmethod``、および ``@classmethod`` も含まれます。

プロジェクトごとの厳格な実施
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``@override`` は、チェッカーが親クラスをオーバーライドするメソッドにデコレーターを使用することを要求する厳格なモードに開発者がオプトインできるようにする場合に最も役立ちます。 厳格な実施は後方互換性のためにオプトインであるべきです。
