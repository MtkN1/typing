.. _historical:

歴史的および非推奨の機能
==========================================================================================

Python 型システムの開発の過程で、型システムを使用しやすくするために Python の文法と標準ライブラリにいくつかの変更が加えられました。 この仕様はすべての例でよりモダンな構文を使用することを目指していますが、型チェッカーは一般的に古い代替案をサポートし、それらを同等と見なすべきです。

このセクションでは、これらのケースをすべて列挙します。

.. _`type-comments`:

型コメント
------------------------------------------------------------------------------------------

変数が特定の型であることを明示的にマークするための一級の構文サポートは、型システムが最初に設計されたときには存在しませんでした。
複雑なケースでの型推論を支援するために、次の形式のコメントを使用できます::

  x = []                # type: list[Employee]
  x, y, z = [], [], []  # type: list[int], list[int], list[str]
  x, y, z = [], [], []  # type: (list[int], list[int], list[str])
  a, b, *c = range(5)   # type: float, float, list[float]
  x = [1, 2]            # type: list[int]

型コメントは、変数定義を含むステートメントの最後の行に配置する必要があります。

これらは、:pep:`526` 変数注釈を使用して変数に注釈を付けることと同等と見なされるべきです::

  x: list[Employee] = []
  x: list[int]
  y: list[int]
  z: list[str]
  x, y, z = [], [], []
  a: float
  b: float
  c: list[float]
  a, b, *c = range(5)
  x: list[int] = [1, 2]

型コメントは ``with`` ステートメントおよび ``for`` ステートメントにも配置でき、コロンの直後に配置されます。

``with`` および ``for`` ステートメントの型コメントの例::

  with frobnicate() as foo:  # type: int
      # ここで foo は int です
      ...

  for x, y in points:  # type: float, float
      # ここで x と y は float です
      ...

スタブでは、変数の存在を宣言するが初期値を与えないことが有用な場合があります。 これは :pep:`526` 変数注釈構文を使用して行うことができます::

  from typing import IO

  stream: IO[str]

上記の構文は、すべてのバージョンの Python のスタブで許容されます。 ただし、Python 3.5 以前のバージョンの非スタブ コードでは、特別なケースがあります::

  from typing import IO

  stream = None  # type: IO[str]

型チェッカーはこれについて文句を言うべきではありません (与えられた型と一致しない値 ``None`` にもかかわらず)、推論された型を ``... | None`` に変更するべきではありません。 ここでの仮定は、他のコードが変数に適切な型の値を与えることを保証し、すべての使用が変数が与えられた型を持っていると仮定できるということです。

.. _`type-comments-function`:

関数定義の型コメント
------------------------------------------------------------------------------------------

一部のツールは、Python 2.7 と互換性がある必要があるコードで型注釈をサポートしたいと考えています。 この目的のために、関数注釈を ``# type:`` コメントに配置できます。 このようなコメントは、関数ヘッダーの直後 (ドキュメント文字列の前) に配置する必要があります。 例: 次の Python 3 コード::

  def embezzle(self, account: str, funds: int = 1000000, *fake_receipts: str) -> None:
      """偽の領収書を使用してアカウントから資金を横領します。"""
      <code goes here>

は次のものと同等です::

  def embezzle(self, account, funds=1000000, *fake_receipts):
      # type: (str, int, *str) -> None
      """偽の領収書を使用してアカウントから資金を横領します。"""
      <code goes here>

メソッドの場合、``self`` の型は必要ありません。

引数のないメソッドの場合は次のようになります::

  def load_cache(self):
      # type: () -> bool
      <code>

引数の型をまだ指定せずに関数またはメソッドの戻り値の型を指定したい場合があります。 これを明示的にサポートするために、引数リストを省略記号に置き換えることができます。 例::

  def send_email(address, sender, cc, bcc, subject, body):
      # type: (...) -> bool
      """メール メッセージを送信します。 成功した場合は True を返します。"""
      <code>

長いパラメーター リストがあり、単一の ``# type:`` コメントでその型を指定するのが面倒な場合があります。 このために、引数を 1 行に 1 つずつリストし、引数に関連付けられたコンマの後に ``# type:`` コメントを追加できます (該当する場合)。 戻り値の型を指定するには、省略記号構文を使用します。 戻り値の型を指定する必要はなく、すべての引数に型を指定する必要はありません。 ``# type:`` コメントを含む行には、正確に 1 つの引数を含める必要があります。 最後の引数の型コメント (該当する場合) は、閉じ括弧の前に配置する必要があります。 例::

  def send_email(address,     # type: Union[str, List[str]]
                 sender,      # type: str
                 cc,          # type: Optional[List[str]]
                 bcc,         # type: Optional[List[str]]
                 subject='',
                 body=None    # type: List[str]
                 ):
      # type: (...) -> bool
      """メール メッセージを送信します。 成功した場合は True を返します。"""
      <code>

注意:

- この構文をサポートするツールは、チェックされる Python バージョンに関係なくサポートする必要があります。 これは、Python 2 と Python 3 の両方にまたがるコードをサポートするために必要です。

- 引数または戻り値に型注釈と型コメントの両方を持つことは許可されていません。

- 短い形式 (例: ``# type: (str, int) -> None``) を使用する場合、インスタンス メソッドおよびクラス メソッドの最初の引数を除いて、すべての引数を考慮する必要があります (通常は省略されますが、含めることもできます)。

- 短い形式では、戻り値の型は必須です。 Python 3 では引数や戻り値の型を省略する場合、Python 2 の表記では ``Any`` を使用する必要があります。

- 短い形式を使用する場合、``*args`` および ``**kwds`` については、対応する型注釈の前に 1 つまたは 2 つのアスタリスクを付けます。 (Python 3 の注釈と同様に、ここでの注釈は、特殊な引数値 ``args`` または ``kwds`` として受け取るタプル/辞書の型ではなく、個々の引数値の型を示します。)

- 他の型コメントと同様に、注釈で使用される名前は、注釈を含むモジュールによってインポートまたは定義されている必要があります。

- 短い形式を使用する場合、注釈全体は 1 行である必要があります。

- 短い形式は、閉じ括弧と同じ行に表示されることもあります。 例::

    def add(a, b):  # type: (int, int) -> int
        return a + b

- 誤った場所に配置された型コメントは、型チェッカーによってエラーとしてフラグが立てられます。 必要に応じて、そのようなコメントを 2 回コメントアウトできます。 例えば::

    def f():
        '''ドキュメント文字列'''
        # type: () -> None  # エラー！

    def g():
        '''ドキュメント文字列'''
        # # type: () -> None  # これは OK です

Python 2.7 コードをチェックする場合、型チェッカーは ``int`` 型と ``long`` 型を同等として扱う必要があります。 ``Text`` として型指定されたパラメーターには、``str`` 型と ``unicode`` 型の引数が許容される必要があります。

.. _`pos-only-double-underscore`:

位置専用パラメーター
------------------------------------------------------------------------------------------

一部の関数は、引数を位置専用でのみ受け取り、呼び出し元が引数の名前を使用してその引数をキーワードで提供することを期待しません。 Python 3.8 (:pep:`570`) 以前は、Python には位置専用パラメーターを宣言する方法がありませんでした。

古いバージョンの Python と互換性を保つ必要があるコードで位置専用パラメーターをサポートするために、型チェッカーは次の特別なケースをサポートする必要があります: 名前が ``__`` で始まり ``__`` で終わらないすべてのパラメーターは位置専用と見なされます::

  def f(__x: int, __y__: int = 0) -> None: ...

  f(3, __y__=1)  # この呼び出しは問題ありません。

  f(__x=3)  # この呼び出しはエラーです。

:pep:`570` 構文と一貫して、位置専用パラメーターはキーワード引数を受け入れるパラメーターの後に表示されることはできません。 型チェッカーはこの要件を強制する必要があります::

  def g(x: int, __y: int) -> None: ...  # 型エラー


標準コレクションのジェネリック
------------------------------------------------------------------------------------------

Python 3.9 (:pep:`585`) 以前は、``list`` などの標準ライブラリのジェネリック型は実行時にパラメーター化できませんでした (つまり、``list[int]`` はエラーをスローします)。 したがって、``typing`` モジュールは主要な組み込み型および標準ライブラリ型のジェネリックエイリアスを提供しました (例: ``typing.List[int]``)。

これらのすべてのケースで、型チェッカーはライブラリ型を ``typing`` モジュールのエイリアスと同等として扱う必要があります。 これには次のものが含まれます:

* ``list`` と ``typing.List``
* ``dict`` と ``typing.Dict``
* ``set`` と ``typing.Set``
* ``frozenset`` と ``typing.FrozenSet``
* ``tuple`` と ``typing.Tuple``
* ``type`` と ``typing.Type``
* ``collections.deque`` と ``typing.Deque``
* ``collections.defaultdict`` と ``typing.DefaultDict``
* ``collections.OrderedDict`` と ``typing.OrderedDict``
* ``collections.Counter`` と ``typing.Counter``
* ``collections.ChainMap`` と ``typing.ChainMap``
* ``collections.abc.Awaitable`` と ``typing.Awaitable``
* ``collections.abc.Coroutine`` と ``typing.Coroutine``
* ``collections.abc.AsyncIterable`` と ``typing.AsyncIterable``
* ``collections.abc.AsyncIterator`` と ``typing.AsyncIterator``
* ``collections.abc.AsyncGenerator`` と ``typing.AsyncGenerator``
* ``collections.abc.Iterable`` と ``typing.Iterable``
* ``collections.abc.Iterator`` と ``typing.Iterator``
* ``collections.abc.Generator`` と ``typing.Generator``
* ``collections.abc.Reversible`` と ``typing.Reversible``
* ``collections.abc.Container`` と ``typing.Container``
* ``collections.abc.Collection`` と ``typing.Collection``
* ``collections.abc.Callable`` と ``typing.Callable``
* ``collections.abc.Set`` と ``typing.AbstractSet`` (名前の変更に注意)
* ``collections.abc.MutableSet`` と ``typing.MutableSet``
* ``collections.abc.Mapping`` と ``typing.Mapping``
* ``collections.abc.MutableMapping`` と ``typing.MutableMapping``
* ``collections.abc.Sequence`` と ``typing.Sequence``
* ``collections.abc.MutableSequence`` と ``typing.MutableSequence``
* ``collections.abc.ByteString`` と ``typing.ByteString``
* ``collections.abc.MappingView`` と ``typing.MappingView``
* ``collections.abc.KeysView`` と ``typing.KeysView``
* ``collections.abc.ItemsView`` と ``typing.ItemsView``
* ``collections.abc.ValuesView`` と ``typing.ValuesView``
* ``contextlib.AbstractContextManager`` と ``typing.ContextManager`` (名前の変更に注意)
* ``contextlib.AbstractAsyncContextManager`` と ``typing.AsyncContextManager`` (名前の変更に注意)
* ``re.Pattern`` と ``typing.Pattern``
* ``re.Match`` と ``typing.Match``

``typing`` モジュールのジェネリックエイリアスは非推奨と見なされ、型チェッカーはそれらが使用された場合に警告を発する可能性があります。

.. _`typing-union`:
.. _`typing-optional`:

``Union`` と ``Optional``
------------------------------------------------------------------------------------------

Python 3.10 (:pep:`604`) 以前は、型の結合を作成するための ``|`` 演算子はサポートされていませんでした。 したがって、``typing.Union`` :term:`special form` も型の結合を作成するために使用できます。 型チェッカーはこれら 2 つの形式を同等として扱う必要があります。

さらに、``Optional`` :term:`special form` は ``None`` との結合と同等です。

例:

* ``int | str`` は ``Union[int, str]`` と同じです
* ``int | str | range`` は ``Union[int, str, range]`` と同じです
* ``int | None`` は ``Optional[int]`` および ``Union[int, None]`` と同じです

.. _`unpack-pep646`:

``Unpack``
------------------------------------------------------------------------------------------

:pep:`646` は、Python 3.11 に ``TypeVarTuple`` を導入し、可変長ジェネリックの使用をサポートするために 2 つの文法変更を行い、インデックス操作および ``*args`` 注釈で ``*`` 演算子の使用を許可しました。 ``Unpack[]`` 演算子は、古いバージョンの Python で同等のセマンティクスをサポートするために追加されました。 これは ``*`` 構文と同等として扱われるべきです。 特に、次のものは同等です:

* ``A[*Ts]`` は ``A[Unpack[Ts]]`` と同じです
* ``def f(*args: *Ts): ...`` は ``def f(*args: Unpack[Ts]): ...`` と同じです
