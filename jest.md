# Jest

## マッチャ

### 文字列型の正規表現との一致判定

文字列型の正規表現との一致判定にはtoMatchマッチャを使用します。

```tsx
test('正規表現による比較', () => {
  expect(essayOnTheBestFlavor()).toMatch(/grapefruit/)
  expect(essayOnTheBestFlavor()).toMatch(new RegExp('grapefruit'))
})
```

このマッチャを用いることで、対象の文字列が特定の正規表現とマッチしているかを判定できます。

### 数値型の判定

整数同士の比較であることが確定している場合は、toBeマッチャを使用します。

```tsx
test('ouncesは12', () => {
  expect(can.ounces).toBe(12)
})
```

浮動小数点型の値を比較する場合、JavaScriptやTypescriptの計算上の問題で小さな誤差が生じます。浮動小数点型の比較の場合は、代わりにtoBeCloseToマッチャを使用します。

```tsx
test('計算時の誤差を打ち消すように比較する', () => {
  expect(0.2 + 0.1).toBeCloseTo(0.3, 5)
})
```

toBeCloseToの第二引数には小数点以下の有効桁数を設定でき、その範囲で値を比較できます。

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

オブジェクトの値が再帰的にすべて一致しているかどうかを判定する際toBeマッチャでは参照先を比較して失敗するので、深い等価判定ができるtoEqualマッチャを使用します。

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

オブジェクト型のパラメータとして持っている浮動小数点型の値をtoEqualで比較する際、計算が含まれていると誤差の問題が発生します。その場合、そのパラメータだけを参照して比較する必要があります。

```tsx
const can1 = {
  flavor: 'grapefruit',
  ounces: 0.1 + 0.2,
}
const can2 = {
  flavor: 'grapefruit',
  ounces: expect.any(Number) as Number,
}

test('オブジェクト型の浮動小数点型のパラメータに計算が入った値が代入された時は、個別に確認する必要がある', () => {
  expect(can1).not.toEqual(can2)
  expect(can1.ounces).toBeCloseTo(can2.ounces, 5)
})
```

オブジェクト型の完全一致ではもう1つtoStrictEqualマッチャというものがあります。こちらはtoEqualの判定に加え、さらに型まで一致しているかどうかを判定します。

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

オブジェクトを比較する際に、データの一部がタイムスタンプ等動的なデータだった場合、単純にtoEqualを使っただけでは比較ができません。expect.anyを比較用のオブジェクトの該当する部分で使用することで、そこにnull、undefined以外でかつインスタンスの型が一致していればパスするようになります。

```tsx
expect(obj).toEqual({
  id: 1,
  name: 'hoge',
  // Numberインスタンスであればパスする
  createdAt:  expect.any(Number) as Number,
})
```

### null/undefined 判定

nullやundefinedの判定には専用のtoBeNullマッチャとtoBeUndefinedマッチャを使用します。

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

falsyな値かどうかを判定するtoBeFalsyマッチャもありますが、これだと0や falseや空文字でもテストを通過してしまうので適切ではありません。

値がnullかundefinedなら通過するテストを記述する場合は、expect.anythingを使ってnot.toEqualを行います。
```tsx
test('nullかundefined', () => {
expect(null).not.toEqual(expect.anything())
expect(undefined).not.toEqual(expect.anything())
expect(1).toEqual(expect.anything())
})
```

## モック化

### モジュールのモック化

jest.mock()を使用することで、モジュールを丸ごとモック化できます。

```tsx
jest.mock('node-fetch')
import fetch, { Response } from 'node-fetch'
```

これは、テストコードで使われているコンポーネントの中で使われている該当のモジュールもモックされます。

jest.requireActual()を使用することで、モジュールの一部を実際の実装と同じ物にもできます。

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

クラスもjest.mockでモック化できます。以下のように引数にモジュールのパスのみを設定した場合は、自動モックになります。この場合、クラス内のメソッドは全てundefindを返すメソッドに置き換わります。

```tsx
import SoundPlayer from './sound-player'
import SoundPlayerConsumer from './sound-player-consumer'
jest.mock('./sound-player')
```

クラスのメソッドがundefindではない値を返すようにしたい場合は、3つある方法の中から適切なものを選択します。

#### プロジェクト上の全てのテストで使うモッククラス

「\_\_mocks\_\_」ディレクトリ内にモックした実装を定義できます。これを使用すると全てのテストファイル上で該当するモックを使用できます。

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

#### ファイル内でのみ使うモック

jest.mock()でモジュールファクトリー引数に生成するモックを定義する高階関数を設定することで、実際にテストでコンストラクタがここで定義したものに置き換わります。使用するファイルが限定されているか、ファイルごとに別の実装を定義したい場合に使用します。

```tsx
import SoundPlayer from './sound-player'
const mockPlaySoundFile = jest.fn()
jest.mock('./sound-player', () => jest.fn().mockImplementation(() => ({ playSoundFile: mockPlaySoundFile })))
```

#### ファイル内でのみ使うモッククラス

mockImplementation() (またはmockImplementationOnce())を使用することで、既存のモックを書き換えることができます。テストごとに実装を変更したい場合に使用します。

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

### Web API のモック化

Reactのfetchメソッドを使用して接続しているWeb APIのモック化を行うには、global.fetchをモック化することで実現できます。

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

実際の実装ではwindow.fetchを使用していますが、Jestが動作するNode.jsには存在しないので、代わりにglobal.fetchを使用します。

### 非同期処理が正常に終了することを検証

expectの.resolvesマッチャを使用します。そのPromiseが解決するまで待機し、さらにその値をマッチングできます。
Promiseがrejectした場合、テストは自動的に失敗します。

```tsx
test('プロミスはlemonを返す', () => expect(Promise.resolve('lemon')).resolves.toBe('lemon'))
```

また、async/awaitを.resolvesと組み合わせて使うことができます。

```tsx
test('プロミスはlemonを返す', async () => {
  await expect(Promise.resolve('lemon')).resolves.toBe('lemon')
  await expect(Promise.resolve('lemon')).resolves.not.toBe('octopus')
})
```

returnは記述しなければ処理を待たずに終了してしまうので省略できません。

### 非同期処理でthrowが返されることを検証

expectの.rejectsマッチャを使用します。そのPromiseが解決するまで待機し、さらにその値をマッチングできます。
Promiseが成功した場合、テストは自動的に失敗します。

```tsx
test('リジェクトしてoctopusを返す', () => expect(Promise.reject(new Error('octopus'))).rejects.toThrow('octopus'))
```

またasync/awaitをrejectsと組み合わせて使うことができます。

```tsx
test('リジェクトしてoctopusを返す', async () =>
  await expect(Promise.reject(new Error('octopus'))).rejects.toThrow('octopus'))
```

## テストをまとめる

### テストの単位

1つのテスト項目はtestメソッドで定義します。

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

エイリアスにitが存在しています。英文法での意味的な使い分けが推奨されていますが、扱いづらいので全て「test」に統一します。

### テスト対象ごとにテストをまとめる

テスト対象が同じものや意味的に近いテストはdescribeメソッドでまとめます。

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

test単体やdescribeレベルのまとまりでテストを実行できるので、階層化することでよりテスト効率を向上させることができます。

### 複数のパターンをテストする

処理が同じで使用する変数のだけ違うテストはtest.eachを用いて1つにまとめます。

```tsx
test.each([
  { a: 1, b: 1, expected: 2 },
  { a: 1, b: 2, expected: 3 },
  { a: 2, b: 1, expected: 3 },
])('.add($a, $b)', ({ a, b, expected }) => {
  expect(a + b).toBe(expected)
})
```

これを有効的に使用することにより、記述するコード量を大幅に減らすことができます。
