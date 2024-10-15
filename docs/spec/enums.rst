列挙型
==========================================================================================

``enum.Enum`` クラスは、型チェッカーで特別なケースの処理を必要とするいくつかの点で他の Python クラスとは異なる動作をします。 このセクションでは、型チェッカーがサポートすべき Enum の動作と、オプションでサポートされる動作について説明します。 ライブラリおよび型スタブの作成者は、これらの動作が一部の型チェッカーでサポートされない可能性があるため、オプションの動作を使用しないことをお勧めします。

列挙型の定義
------------------------------------------------------------------------------------------

Enum クラスは、「クラス構文」または「関数構文」を使用して定義できます。 関数構文は、個々の引数として渡される名前、名前のリストまたはタプル、カンマ区切りまたはスペース区切りの名前の文字列、名前/値のペアを含むリストまたはタプル、および名前/値のアイテムの辞書など、列挙メンバーを指定するいくつかの方法を提供します。

型チェッカーはクラス構文をサポートする必要がありますが、関数構文（そのさまざまな形式）はオプションです::

    class Color1(Enum): # サポートされている
        RED = 1
        GREEN = 2
        BLUE = 3

    Color2 = Enum('Color2', 'RED', 'GREEN', 'BLUE')  # オプション
    Color3 = Enum('Color3', ['RED', 'GREEN', 'BLUE'])  # オプション
    Color4 = Enum('Color4', ('RED', 'GREEN', 'BLUE'))  # オプション
    Color5 = Enum('Color5', 'RED, GREEN, BLUE')  # オプション
    Color6 = Enum('Color6', 'RED GREEN BLUE')  # オプション
    Color7 = Enum('Color7', [('RED', 1), ('GREEN', 2), ('BLUE', 3)])  # オプション
    Color8 = Enum('Color8', (('RED', 1), ('GREEN', 2), ('BLUE', 3)))  # オプション
    Color9 = Enum('Color9', {'RED': 1, 'GREEN': 2, 'BLUE': 3})  # オプション

Enum クラスは、``enum.Enum`` のサブクラスまたは ``enum.EnumType``（またはそのサブクラス）をメタクラスとして使用する任意のクラスを使用して定義することもできます。 ``enum.EnumType`` は Python 3.11 以前では ``enum.EnumMeta`` と呼ばれていました。 型チェッカーはそのようなクラスを列挙型として扱う必要があります::

    class CustomEnum1(Enum):
        pass

    class Color7(CustomEnum1):  # サポートされている
        RED = 1
        GREEN = 2
        BLUE = 3

    class CustomEnumType(EnumType):
        pass

    class CustomEnum2(metaclass=CustomEnumType):
        pass

    class Color8(CustomEnum2):  # サポートされている
        RED = 1
        GREEN = 2
        BLUE = 3

列挙型の動作
------------------------------------------------------------------------------------------

Enum クラスは反復可能でインデックス付け可能であり、値を使用して呼び出すことができ、その値を持つ列挙メンバーを検索します。 型チェッカーはこれらの動作をサポートする必要があります::

    class Color(Enum):
        RED = 1
        GREEN = 2
        BLUE = 3

    for color in Color:
        reveal_type(color)  # 明らかにされた型は 'Color' です

    reveal_type(Color["RED"])  # 明らかにされた型は 'Literal[Color.RED]'（または 'Color'）です
    reveal_type(Color(3))  # 明らかにされた型は 'Literal[Color.BLUE]'（または 'Color'）です

ほとんどの Python クラスとは異なり、列挙型クラスを呼び出してもそのコンストラクタは呼び出されません。 代わりに、その呼び出しは列挙メンバーの値ベースの検索を実行します。

1 つ以上のメンバーが定義されている Enum クラスはサブクラス化できません。 それらは暗黙的に「最終的」です。 型チェッカーはこれを強制する必要があります::

    class EnumWithNoMembers(Enum):
        pass

    class Shape(EnumWithNoMembers):  # OK（メンバーが定義されていないため）
        SQUARE = 1
        CIRCLE = 2

    class ExtendedShape(Shape):  # 型チェッカーエラー: Shape は暗黙的に最終的です
        TRIANGLE = 3

メンバーの定義
------------------------------------------------------------------------------------------

「クラス構文」を使用する場合、Enum クラスはメンバーとその他の（メンバー以外の）属性の両方を定義できます。 ``EnumType`` メタクラスは、メンバーと非メンバーを区別する一連のルールを適用します。 型チェッカーは、これらのルールの中で最も一般的なものを尊重する必要があります。 あまり使用されないルールはオプションです。 動的な値が使用される場合、これらのルールの一部は静的に評価および強制することが不可能な場合があります。

* 属性が型注釈付きでクラス本体に定義されているが、値が割り当てられていない場合、型チェッカーはこれが非メンバー属性であると見なす必要があります::

    class Pet(Enum):
        genus: str  # 非メンバー属性
        species: str  # 非メンバー属性

        CAT = 1  # メンバー属性
        DOG = 2  # メンバー属性

  型スタブ内では、メンバーは実際のランタイム値を使用して定義することも、プレースホルダーとして ``...`` を使用することもできます::

    class Pet(Enum):
        genus: str  # 非メンバー属性
        species: str  # 非メンバー属性

        CAT = ...  # メンバー属性
        DOG = ...  # メンバー属性

* Enum クラス内で定義されたメンバーには明示的な型注釈を含めるべきではありません。 型チェッカーはすべてのメンバーに対してリテラル型を推論する必要があります。 型チェッカーは、列挙メンバーに型注釈が使用されている場合、これは誤ったものであり、コードの読者にとって誤解を招くため、エラーを報告する必要があります::

    class Pet(Enum):
        CAT = 1  # OK
        DOG: int = 2  # 型チェッカーエラー

* クラス内で定義されたメソッド、呼び出し可能なもの、デスクリプタ（プロパティを含む）、およびネストされたクラスは、``EnumType`` メタクラスによって列挙メンバーとして扱われず、型チェッカーによっても列挙メンバーとして扱われるべきではありません::

    def identity(x): return x

    class Pet(Enum):
        CAT = 1  # メンバー属性
        DOG = 2  # メンバー属性

        converter = lambda x: str(x)  # 非メンバー属性
        transform = identity  # 非メンバー属性

        @property
        def species(self) -> str:  # 非メンバー属性
            return "mammal"

        def speak(self) -> None:  # 非メンバー属性
            print("meow" if self is Pet.CAT else "woof")

        class Nested: ... # 非メンバー属性

* 同じ列挙の別のメンバーの値が割り当てられている属性は、それ自体がメンバーではありません。 代わりに、最初のメンバーのエイリアスです::

    class TrafficLight(Enum):
        RED = 1
        GREEN = 2
        YELLOW = 3

        AMBER = YELLOW  # YELLOW のエイリアス

    reveal_type(TrafficLight.AMBER)  # 明らかにされた型は Literal[TrafficLight.YELLOW] です

* Python 3.11 以降を使用している場合、``enum.member`` および ``enum.nonmember`` クラスを使用して、メンバーと非メンバーを明確に区別できます。 型チェッカーはこれらのクラスをサポートする必要があります::

    class Example(Enum):
        a = member(1)  # メンバー属性
        b = nonmember(2)  # 非メンバー属性

        @member
        def c(self) -> None:  # メンバー属性
            pass

    reveal_type(Example.a)  # 明らかにされた型は Literal[Example.a] です
    reveal_type(Example.b)  # 明らかにされた型は int または Literal[2] です
    reveal_type(Example.c)  # 明らかにされた型は Literal[Example.c] です

* プライベート名（ダブルアンダースコアで始まり、ダブルアンダースコアで終わらない）を持つ属性は、非メンバーとして扱われます::

    class Example(Enum):
        A = 1  # メンバー属性
        __B = 2  # 非メンバー属性

    reveal_type(Example.A)  # 明らかにされた型は Literal[Example.A] です
    reveal_type(Example.__B)  # 型エラー: プライベート名がマングルされています

* 列挙クラスは ``_ignore_`` という名前のクラスシンボルを定義できます。 これは、列挙クラスから削除される名前のリストまたはスペース区切りの名前のリストを含む文字列である可能性があります。 型チェッカーはこのメカニズムをサポートする場合があります::

    class Pet(Enum):
        _ignore_ = "DOG FISH"
        CAT = 1  # メンバー属性
        DOG = 2  # 一時変数、最終的な列挙クラスから削除されます
        FISH = 3  # 一時変数、最終的な列挙クラスから削除されます

メンバー名
------------------------------------------------------------------------------------------

すべての列挙メンバーオブジェクトには、メンバーの名前を含む ``_name_`` 属性があります。 また、同じ名前を返す ``name`` プロパティもあります。 型チェッカーはメンバーの名前のリテラル型を推論する場合があります::

    class Color(Enum):
        RED = 1
        GREEN = 2
        BLUE = 3

    reveal_type(Color.RED._name_)  # 明らかにされた型は Literal["RED"]（または str）です
    reveal_type(Color.RED.name)  # 明らかにされた型は Literal["RED"]（または str）です

    def func1(red_or_blue: Literal[Color.RED, Color.BLUE]):
        reveal_type(red_or_blue.name)  # 明らかにされた型は Literal["RED", "BLUE"]（または str）です

    def func2(any_color: Color):
        reveal_type(any_color.name)  # 明らかにされた型は Literal["RED", "BLUE", "GREEN"]（または str）です

メンバーの値
------------------------------------------------------------------------------------------

すべての列挙メンバーオブジェクトには、メンバーの値を含む ``_value_`` 属性があります。 また、同じ値を返す ``value`` プロパティもあります。 型チェッカーはメンバーの値の型を推論する場合があります::

    class Color(Enum):
        RED = 1
        GREEN = 2
        BLUE = 3

    reveal_type(Color.RED._value_)  # 明らかにされた型は Literal[1]（または int または object または Any）です
    reveal_type(Color.RED.value)  # 明らかにされた型は Literal[1]（または int または object または Any）です

    def func1(red_or_blue: Literal[Color.RED, Color.BLUE]):
        reveal_type(red_or_blue.value)  # 明らかにされた型は Literal[1, 2]（または int または object または Any）です

    def func2(any_color: Color):
        reveal_type(any_color.value)  # 明らかにされた型は Literal[1, 2, 3]（または int または object または Any）です

``_value_`` の値はコンストラクタメソッドで割り当てることができます。 この手法は、メンバーの値と非メンバー属性の両方を初期化するために使用されることがあります。 クラス本体で割り当てられた値がタプルである場合、アンパックされたタプル値がコンストラクタに渡されます。 型チェッカーは、割り当てられたタプル値とコンストラクタのシグネチャの一貫性を検証する場合があります::

    class Planet(Enum):
        def __init__(self, value: int, mass: float, radius: float):
            self._value_ = value
            self.mass = mass
            self.radius = radius

        MERCURY = (1, 3.303e+23, 2.4397e6)
        VENUS = (2, 4.869e+24, 6.0518e6)
        EARTH = (3, 5.976e+24, 6.37814e6)
        MARS = (6.421e+23, 3.3972e6)  # 型チェッカーエラー（オプション）
        JUPITER = 5  # 型チェッカーエラー（オプション）

    reveal_type(Planet.MERCURY.value)  # 明らかにされた型は Literal[1]（または int または object または Any）です

列挙クラス内で ``enum.auto`` クラスおよび ``_generate_next_value_`` メソッドを使用して、列挙メンバーの値を自動的に生成できます。 型チェッカーはこれらをサポートしてメンバー値のリテラル型を推論する場合があります::

    class Color(Enum):
        RED = auto()
        GREEN = auto()
        BLUE = auto()

    reveal_type(Color.RED.value)  # 明らかにされた型は Literal[1]（または int または object または Any）です

列挙クラスが ``_value_`` に対して明示的な型注釈を提供する場合、型チェッカーは値が ``_value_`` に割り当てられるときにこの宣言された型を強制する必要があります::

    class Color(Enum):
        _value_: int
        RED = 1 # OK
        GREEN = "green"  # 型エラー

    class Planet(Enum):
        _value_: str

        def __init__(self, value: int, mass: float, radius: float):
            self._value_ = value # 型エラー

        MERCURY = (1, 3.303e+23, 2.4397e6)

列挙メンバーのリテラル値が提供されていない場合（型スタブファイル内では提供されていないことがよくあります）、型チェッカーは ``_value_`` 属性の型を使用できます::

    class ColumnType(Enum):
        _value_: int
        DORIC = ...
        IONIC = ...
        CORINTHIAN = ...

    reveal_type(ColumnType.DORIC.value)  # 明らかにされた型は int（または object または Any）です

列挙リテラルの展開
------------------------------------------------------------------------------------------

型システムの観点から、ほとんどの列挙クラスはその列挙内のリテラルメンバーの共用体と同等です。 （このルールは、フラグを任意の方法で組み合わせることができるため、``enum.Flag`` から派生したクラスには適用されません。）列挙クラスとその列挙内のリテラルメンバーの共用体の間には同等性があるため、2 つの型は互換的に使用できます。 したがって、型チェッカーは、型の絞り込みおよび枯渇検出中に列挙型（``enum.Flag`` から派生していない）をリテラル値の共用体に展開する場合があります::

    class Color(Enum):
        RED = 1
        GREEN = 2
        BLUE = 3

    def print_color1(c: Color):
        if c is Color.RED or c is Color.BLUE:
            print("red or blue")
        else:
            reveal_type(c)  # 明らかにされた型は Literal[Color.GREEN] です

    def print_color2(c: Color):
        match c:
            case Color.RED | Color.BLUE:
                print("red or blue")
            case Color.GREEN:
                print("green")
            case _:
                reveal_type(c)  # 明らかにされた型は Never です

同様に、型チェッカーはすべてのリテラルメンバーの完全な共用体を列挙型と同等として扱う必要があります::

    class Answer(Enum):
        Yes = 1
        No = 2

    def func(val: object) -> Answer:
        if val is not Answer.Yes and val is not Answer.No:
            raise ValueError("Invalid value")
        reveal_type(val)  # 明らかにされた型は Answer（または Literal[Answer.Yes, Answer.No]）です
        return val  # OK
