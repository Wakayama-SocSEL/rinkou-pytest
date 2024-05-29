# 2　テスト関数を書く

## 2.1　サンプルアプリケーションをインストールする

- 本章ではkanbanアプリCardsのテストを作成する
- Cardsプロジェクトはインストール可能なパッケージである

```
# Cardsをインストールする
cd ./code
pip install ./cards_proj
```

- これで```import cards```のようにテストがアプリを認識できる

```
# Cardsを試してみる
cards add do something --owner Brian
cards add do something else
cards

cards update 2 --owner Brian
cards

cards start 1
cards finish 1
cards start 2
cards

cards delete 1
cards
```

- Cardsにはadd、update、start、finish、delete、一覧表示機能がある

## 2.2　テストを書きながら学ぶ

- CardsはCLI、API、DBの3層に分かれる
- CLI-API間のやり取りはCardというデータクラスを使う

```python
# cards_proj/src/cards/api.py

@dataclass
class Card:
    summary: str = None
    owner: str = None
    state: str = "todo"
    id: int = field(default=None, compare=False)

    @classmethod
    def from_dict(cls, d):
        return Card(**d)
    def to_dict(self):
        return asdict(self)
```

```
# データクラスを理解するためのテストを実行する
cd ./ch2
pytest test_card.py
```

## 2.3　assert文を使う

- pytestでは、主にassert文でテストが失敗したことを伝える
- Pythonの仕様内の機能を使うので、pytestは非常にシンプルといえる

| pytest               | unittest              |
| -------------------- | --------------------- |
| assert a             | assertTrue(a)         |
| assert not a         | assertFalse(a)        |
| assert a == b        | assertEqual(a, b)     |
| assert a != b        | assertNotEqual(a, b)  |
| assert a is None     | assertIsNone(a)       |
| assert a is not None | assertIsNotNone(a)    |
| assert a <= b        | assertLessEqual(a, b) |
| ...                  | ...                   |

- pytestには「assertの書き換え」と呼ばれる機能がある
- assertの呼び出しをインターセプトし、アサーションが失敗した理由をさらに詳しく説明できる何かに置き換える

```
# 失敗するテストを実行し、出力を確認する
pytest -vv test_card_fail.py
```

```
# Pythonの通常の出力と比較する
python test_card_fail.py
```

## 2.4　pytest.fail()と例外でテストを失敗させる

- pytest.fail()を使ってもテストを失敗させられる

```
# pytest.fail()を使ってテストを失敗させる
pytest test_alt_fail.py
```

## 2.5　アサーションヘルパー関数を書く

- アサーションヘルパーは複雑なアサーションチェックの作成に使われる関数である
- 例えばCardデータクラスはIID以外の値が同じであれば同じとみなされるので、assert_identicalヘルパー関数を作成してより厳格なチェックを行う

```
# ヘルパー関数を実行する
pytest test_helper.py
```

## 2.6　想定される例外をテストする

- 想定されている例外をテストするにはpytest.raises()を使う
- 例えばパス引数を要求するCardsDBクラスにパスを渡さなかった場合はTypeErrorを起こす

```python
# ch2/test_exceptions.py

def test_no_path_raises():
    with pytest.raises(TypeError):
        cards.CardsDB()
```

- with節でTypeError例外が発生しなかったり、他の例外が発生したりするとテストは失敗する
- メッセージが正しいかどうかもチェックできる

```python
# ch2/test_exceptions.py

def test_raises_with_info():
    match_regex = "missing 1 .* positional argument"
    with pytest.raises(TypeError, match=match_regex):
        cards.CardsDB()


def test_raises_with_info_alt():
    with pytest.raises(TypeError) as exc_info:
        cards.CardsDB()
    expected = "missing 1 required positional argument"
    assert expected in str(exc_info.value)
```

## 2.7　テスト関数を構造化する

- テストは準備-実行-検証（Arrange-Act-Assert / Given-When-Then）の順に記述する

```python
# ch2/test_structure.py

def test_to_dict():
    # GIVEN a Card object with known contents
    c1 = Card("something", "brian", "todo", 123)

    # WHEN we call to_dict() on the object
    c2 = c1.to_dict()

    # THEN the result will be a dictionary with known content
    c2_expected = {
        "summary": "something",
        "owner": "brian",
        "state": "todo",
        "id": 123,
    }
    assert c2 == c2_expected
```

## 2.8　テストをクラスにまとめる

- pytestではテストをクラスにまとめることもできる

```python
# ch2/test_classes.py

class TestEquality:
    def test_equality(self):
        c1 = Card("something", "brian", "todo", 123)
        c2 = Card("something", "brian", "todo", 123)
        assert c1 == c2
    def test_equality_with_diff_ids(self):
        c1 = Card("something", "brian", "todo", 123)
        c2 = Card("something", "brian", "todo", 4567)
        assert c1 == c2

    def test_inequality(self):
        c1 = Card("something", "brian", "todo", 123)
        c2 = Card("completely different", "okken", "done", 123)
        assert c1 != c2
```

## 2.9 テストの一部を実行する

```
# 1つ上のディレクトリに戻る
cd ..
```

```
# テストメソッド、テストクラス、モジュールを1つだけ実行する
pytest ch2/test_classes.py::TestEquality::test_equality
pytest ch2/test_classes.py::TestEquality
pytest ch2/test_classes.py
```

```
# テスト関数、モジュールを1つだけ実行する
pytest ch2/test_card.py::test_defaults
pytest ch2/test_card.py
```

```
# ディレクトリ全体を実行する
pytest ch2
```

- -kフラグは引数として式をとり、名前にその式とマッチする部分文字列が含まれるテストを実行する

```
# TestEqualityクラスを実行する
pytest -v -k TestEquality
pytest -v -k TestEq
```

```
# 名前にequalityを含むテストをすべて実行する
pytest -v --tb=no -k equality
```

- and、not、orと()を使って複雑な式を作ることもできる

```
# TestEqualityクラスを除く、名前にdictかidsを含むテストをすべて実行する
pytest -v --tb=no -k "(dict or ids) and not TestEquality"
```
