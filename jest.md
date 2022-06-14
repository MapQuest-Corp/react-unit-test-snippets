# Jest

## 共通

### 文字列型の正規表現との一致判定

文字列型の正規表現との一致判定には toMatch マッチャを使用します。

```tsx
test('正規表現による比較', () => {
  expect(essayOnTheBestFlavor()).toMatch(/grapefruit/)
  expect(essayOnTheBestFlavor()).toMatch(new RegExp('grapefruit'))
})
```

このマッチャを用いることで、対象の文字列が特定の正規表現とマッチしているかを判定することが出来ます。

### 数値型の判定

比較する値が整数同士であることが確定している場合は、toBe マッチャを使用します。

```tsx
test('ouncesは12', () => {
  expect(can.ounces).toBe(12)
})
```

浮動小数点型の値を比較する場合、JavaScript や Typescript の計算上の問題で限りなく小さな誤差で値が一致しないことがあります。浮動小数点型の比較の場合は、代わりに toBeCloseTo マッチャを使用します。

```tsx
test('計算時の誤差を打ち消すように比較する', () => {
  expect(0.2 + 0.1).toBeCloseTo(0.3, 5)
})
```

toBeCloseTo の第二引数には小数点以下の有効桁数を設定することができ、その範囲で値を比較出来ます。

<!--
バージョン27.x～
オブジェクト型のインスタンス内での比較でも上記と同じ問題が発生するので、expect.closeTo を使用することで同じように比較することが出来るようになります。

```tsx
test('compare float in object properties', () => {
  expect({
    title: '0.1 + 0.2',
    sum: 0.1 + 0.2,
  }).toEqual({
    title: '0.1 + 0.2',
    sum: expect.closeTo(0.3, 5),
  })
})
``` -->

### オブジェクト型の完全一致判定

オブジェクトの値が再帰的にすべて一致しているかどうかを判定する際 toBe マッチャでは参照先を比較して失敗するので、深い等価判定ができる toEqual マッチャを使用します。

```tsx
const can1 = {
  flavor: 'grapefruit',
  ounces: 12,
}
const can2 = {
  flavor: 'grapefruit',
  ounces: 12,
}

test('オブジェクト型同士の深い比較', () => {
  expect(can1).toEqual(can2)
})
test('toBeではオブジェクト型同士の深い比較は出来ない', () => {
  expect(can1).not.toBe(can2)
})
```

オブジェクト型のパラメータとして持っている浮動小数点型の値を toEqual で比較した際、計算を行っている場合数値型の判定で説明した問題により失敗するので、そのパラメータだけを参照して比較する必要があります。

```tsx
const can1 = {
  flavor: 'grapefruit',
  ounces: 0.1 + 0.2,
}
const can2 = {
  flavor: 'grapefruit',
  ounces: 0.3,
}

test('オブジェクト型の浮動小数点型のパラメータに計算が入った値が代入された時は、個別に確認する必要がある', () => {
  expect(can1).not.toEqual(can2)
  expect(can1.ounces).toBeCloseTo(can2.ounces, 5)
})
```

オブジェクト型の完全一致ではもう 1 つ toStrictEqual マッチャというものがあります。こちらは toEqual の判定に加え、さらに型まで一致しているかどうかを判定します。

```tsx
class LaCroix {
  constructor(flavor: string) {
    this.flavor = flavor
  }

  flavor: string
}

test('使用している型まで比較', () => {
  expect(new LaCroix('レモン')).toEqual({ flavor: 'レモン' })
  expect(new LaCroix('レモン')).not.toStrictEqual({ flavor: 'レモン' }) // 使用している型が違うので一致しない
})
```

### オブジェクト型の部分一致判定

オブジェクトを比較する際に、データの一部がタイムスタンプ等動的なデータだった場合、単純に toEqual を使っただけでは比較ができない。expect.any を比較用のオブジェクトの該当する部分で使用することで、そこに null、undefined 以外でかつインスタンスの型が一致していればパスするようになる。

```tsx
expect(obj).toEqual({
  id: 1,
  name: 'hoge',
  // Numberインスタンスであればパスする
  createdAt: expect.any(Number) as number,
})
```

### null/undefined 判定

null や undefined の判定には専用の toBeNull マッチャと toBeUndefined マッチャを使用します。

```tsx
function bloop() {
  return null
}

test('null判定', () => {
  expect(bloop()).toBeNull()
})
```

```tsx
test('undefined判定', () => {
  expect(bestDrinkForFlavor('octopus')).toBeUndefined()
})
```

falsy な値かどうかを判定する toBeFalsy マッチャもありますが、これだと 0 や false や空文字でもテストを通過してしまうので適切ではありません。

### モジュールのモック化

jest.mock()を使用することで、モジュールを丸ごとモック化することが出来ます。

```tsx
jest.mock('node-fetch')
import fetch, { Response } from 'node-fetch'
```

これは、テストコードで使われているコンポーネントの中で使われている該当のモジュールもモックされます。

jest.requireActual()を使用することで、モジュールの一部を実際の実装と同じ物にすることもできます。

```tsx
jest.mock('node-fetch')
import fetch from 'node-fetch'
const {Response} = jest.requireActual('node-fetch')

test('Responseは元の実装のまま使用できる', async () => {
  fetch.mockReturnValue(Promise.resolve(new Response('4')))
  ・
  ・
  ・
})
```

### クラスのモック化

クラスも jest.mock でモック化することが出来ます。以下のように引数にモジュールのパスのみを設定した場合は、自動モックになります。この場合、クラス内のメソッドは全て undefind を返すメソッドに置き換わります。

```tsx
import SoundPlayer from './sound-player'
import SoundPlayerConsumer from './sound-player-consumer'
jest.mock('./sound-player')
```

クラスのメソッドが undefind ではない値を返すようにしたい場合は、3 つある方法の中から適切なものを選択します。

#### マニュアルモック

「\_\_mocks\_\_」ディレクトリ内にモックした実装を定義することができ。これを使用すると全てのテストファイル上で該当するモックを使用することが出来ます。

```tsx
/// __mocks__/sound-player.js
// このファイルをテストファイルにインポートする
export const mockPlaySoundFile = jest.fn()
const mock = jest.fn().mockImplementation(() => ({ playSoundFile: mockPlaySoundFile }))

export default mock
```

```tsx
/// sound-player-consumer.test.js
import SoundPlayer, { mockPlaySoundFile } from './sound-player'
import SoundPlayerConsumer from './sound-player-consumer'
jest.mock('./sound-player')

beforeEach(() => {
  // 全てのインスタンスとコンストラクタおよびすべてのメソッドの呼び出しをクリアする
  SoundPlayer.mockClear()
  mockPlaySoundFile.mockClear()
})
```

#### jest.mock() をモジュールファクトリ引数で呼ぶ

jest.mock()でモジュールファクトリー引数に生成するモックを定義する高階関数を設定することで、実際にテストでコンストラクタがここで定義したものに置き換わります。使用するファイルが限定されているか、ファイルごとに別の実装を定義したい場合に使用します。

```tsx
import SoundPlayer from './sound-player'
const mockPlaySoundFile = jest.fn()
jest.mock('./sound-player', () => jest.fn().mockImplementation(() => ({ playSoundFile: mockPlaySoundFile })))
```

#### mockImplementation() または mockImplementationOnce() を使用したモックを置き換える

mockImplementation() (または mockImplementationOnce())を使用することで、既存のモックを書き換えることが出来ます。テストごとに実装を変更したい場合に使用します。

```tsx
import SoundPlayer from './sound-player'
import SoundPlayerConsumer from './sound-player-consumer'

jest.mock('./sound-player')

describe('When SoundPlayer throws an error', () => {
  beforeAll(() => {
    SoundPlayer.mockImplementation(() => {
      playSoundFile: () => {
        throw new Error('Test error')
      }
    })
  })
})
```

## 非同期処理テスト

### WebAPI のモック化

React の fetch メソッドを使用して接続している WebAPI のモック化を行うには、global.fetch をモック化することで実現できます。

```tsx
global.fetch = jest.fn(
  (url: string, init: RequestInit) =>
    new Promise((resolve, reject) => {
      if (url.endsWith('/data-api/uploads/v1?…')) {
        ・・・
      }
      ・
      ・
      ・
    })
) as jest.Mock
```

実際の実装では windows.fetch を使用していますが、Jest が動作する Node.js には存在しないので、代わりに global.fetch を使用します。

### 非同期処理で結果が resolve になることを検証

expect の.resolves マッチャを使用します。その Promise が解決するまで待機し、さらにその値をマッチングすることが出来ます。
Promise が reject した場合、テストは自動的に失敗します。

```tsx
test('プロミスはlemonを返す', () => expect(Promise.resolve('lemon')).resolves.toBe('lemon'))
```

また、async/await を.resolves と組み合わせて使うことができます。

```tsx
test('プロミスはlemonを返す', async () => {
  await expect(Promise.resolve('lemon')).resolves.toBe('lemon')
  await expect(Promise.resolve('lemon')).resolves.not.toBe('octopus')
})
```

return は記述しなければ処理を待たずに終了してしまうので省略できません。

### 非同期処理で結果が reject になることを検証

expect の.rejects マッチャを使用します。その Promise が解決するまで待機し、さらにその値をマッチングすることが出来ます。
Promise が成功した場合、テストは自動的に失敗します。

```tsx
test('リジェクトしてoctopusを返す', () => expect(Promise.reject(new Error('octopus'))).rejects.toThrow('octopus'))
```

また async/await を rejects と組み合わせて使うことができます。

```tsx
test('リジェクトしてoctopusを返す', async () =>
  await expect(Promise.reject(new Error('octopus'))).rejects.toThrow('octopus'))
```

## テストをまとめる

### テストの単位

1 つのテスト項目は test メソッドで定義します。

```tsx
test('lemonがリストに含まれる', () =>
  fetchBeverageList().then(list => {
    expect(list).toContain('lemon')
    ・
    ・
    ・
  })
)
```

エイリアスに it が存在しています。英文法での意味的な使い分けが推奨されていますが、扱いづらいので全て「test」に統一します。

### テスト対象ごとにテストをまとめる

テスト対象が同じものや意味的に近いテストは describe メソッドでまとめます。

```tsx
describe('持っているのペットボトル飲料の中身', () => {
  test('飲み物は旨い', () => {
    expect(myBeverage.delicious).toBeTruthy()
  })

  test('飲み物は炭酸ではない', () => {
    expect(myBeverage.sour).toBeFalsy()
  })
})
```

test 単体や describe レベルのまとまりでテストを実行することもできるので、階層化することでよりテスト効率を向上させることが出来ます。

### 複数のパターンをテストする

処理が同じで使用する変数のだけ違うテストは test.each を用いて 1 つにまとめます。

```tsx
test.each([
  { a: 1, b: 1, expected: 2 },
  { a: 1, b: 2, expected: 3 },
  { a: 2, b: 1, expected: 3 },
])('.add($a, $b)', ({ a, b, expected }) => {
  expect(a + b).toBe(expected)
})
```

これを有効的に使用することにより、記述するコード量を大幅に減らすことが出来ます。
