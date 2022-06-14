## テンプレート
- [jest.clearAllMocks()](https://jestjs.io/ja/docs/jest-object#jestclearallmocks)  
mock.callsをクリアする。toHaveBeenCalledTimes()やtoHaveBeenCalledTimes()によるアサートが適切に行える。
- [cleanup()](https://testing-library.com/docs/react-testing-library/api/#cleanup)  
コンポーネントをツリーからunmountする。afterEachのタイミングで自動的に呼ばれる。
```tsx
import { getByRole, render, screen, within } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

afterEach(() => {
  jest.clearAllMocks()
})
```

## DOMツリーから要素を見つける（クエリ）

### 使うべきクエリの優先順位
ユーザが見たり触れたりするものに基づいてテストすることを推奨する。上のクエリほど好ましい。
1. getByRole
1. getByLabelText
1. getByPlaceholderText
1. getByText
1. getByDisplayValue
1. getByAltText
1. getByTitle
1. getByTestId

### 表示テキストで要素をクエリする
~ByTextを使う。
getBy~はクエリに当てはまるDOM要素が見つからないか、2つ以上見つかった場合に例外をスローする（catchしなければテストに失敗する）。
screenの範囲はデフォルトでdocument.body。
```tsx
render(<div>Lorem Ipsum</div>)

// 文字列は完全一致
screen.getByText('Lorem Ipsum')
// 正規表現は部分一致
screen.getByText(/Lorem/)
```

### DOM要素のロールで要素をクエリする
~ByRoleを使う。
HTML要素には[暗黙的なroleが設定されている](https://xn--scptdeveloper-o24l8kvx0dwh.mozilla.org/ja/docs/Web/Accessibility/ARIA/Roles)ことも多い。

### inputの値で要素をクエリする
inputに紐づけたラベルを元に~ByLabelTextで取得するのが好ましい。
```tsx
render(
  <label htmlFor="input-number">
    Number
    <input id="input-number" type="number" value="5" />
  </label>
)

// inputの値
expect(screen.getByLabelText('Number')).toHaveValue(5)
//ユーザが見る値
expect(screen.getByLabelText('Number')).toHaveDisplayValue('5')
```

### 画像、アイコンで要素をクエリする
アクセシビリティを元にテストする。
意味的（semantic）画像はrole="img"でtitle要素を含む。アイコンボタンなど操作可能な要素ならaria-label属性を付与する。data-testidはなるべく使わないこと。

```tsx
render(
  <IconButton aria-label="delete">
    <SvgIcon>
      <path d="M20 12l-1.41-1.41L13 16.17V4h-2v12.17l-5.58-5.59L4 12l8 8 8-8z" />
    </SvgIcon>
  </IconButton>
)

screen.getByRole('button', { name: 'delete' })
```

## アサート（Assertion）
Testing Libraryでは、DOM要素をアサートするためにexpect関数が拡張されている。  
https://github.com/testing-library/jest-dom#custom-matchers

### 見た目の色をテストする
MUIの場合、与えられたThemeのPalette色を使っているかテストする。
```tsx
import { Button, createTheme, ThemeProvider } from '@mui/material'

// ...

const colorTheme = createTheme()

render(
  <ThemeProvider theme={colorTheme}>
    <Button variant="contained" color="secondary">
      Secondary Button
    </Button>
  </ThemeProvider>
)

// variant="contained"のButtonのテキスト色はcontrastTextが使われる
const button = screen.getByRole('button')
expect(button).toHaveStyle(`color: ${colorTheme.palette.secondary.contrastText}`)
```

### DOM要素が存在しないことをテストする
queryBy~を使う。
DOM要素が見つからない場合にnullを返す。2つ以上見つかった場合は例外をスローする。

### 非同期に表示されるDOM要素をテストする
findBy~を使う。
アニメーションで遅れて表示される場合や、レンダリングまでにuseEffectが挟まる場合など。
```tsx
render(
  <Tooltip title="Hovered">
    <div>Lorem Ipsum</div>
  </Tooltip>
)

// カーソルホバー前には表示されていないこと
expect(screen.queryByText('Hovered')).toBeNull()

await userEvent.hover(screen.getByText('Lorem Ipsum'))
await screen.findByText('Hovered')
```

### ユーザイベント
指定したイベントのみを発火するfireEventよりも、ユーザの操作に近い形でイベントを発火するuserEventを使う方が好ましい。
どちらもactは不要。
```tsx
const mockOnChange = jest.fn()
render(<input type="text" value="" onChange={mockOnChange} />)

// "Lorem Ipsum" は11文字なのでchangeイベントが11回発火する
await userEvent.type(screen.getByRole('textbox'), 'Lorem Ipsum')
expect(mockOnChange).toBeCalledTimes(11)
```
- ユーザによる連続的な入力
https://testing-library.com/docs/user-event/setup/#starting-a-session-per-setup
```tsx
function setup(jsx: JSX.Element) {
  return {
    user: userEvent.setup(),
    ...render(jsx),
  }
}

test('input text by user', async () => {
  const mockOnChange = jest.fn()
  const { user } = setup(<input type="text" value="" onChange={mockOnChange} />)

  // "Lorem Ipsum" は11文字なのでchangeイベントが11回発火する
  await user.type(screen.getByRole('textbox'), 'Lorem Ipsum')
  expect(mockOnChange).toBeCalledTimes(11)
})
```

## デバッグ
レンダリングされたDOMツリーは`screen.debug()`でコンソール出力できる。

## その他リンク
- https://testing-library.com/docs/react-testing-library/cheatsheet/
- https://testing-playground.com/