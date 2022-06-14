
## テンプレート
```tsx
/* eslint-disable import/no-extraneous-dependencies */
import { act, renderHook } from '@testing-library/react-hooks'
```

## レンダリング
### stateを更新する
actで囲まなくても警告が出るだけでエラーにはならないが、[テストをReactのブラウザでの動作に近づける](https://reactjs.org/docs/test-utils.html#act)目的で必要。
```tsx
const useCounter = () => {
  const [count, setCount] = useState(0)
  const increment = useCallback(() => setCount((x) => x + 1), [])
  return { count, increment }
}

test('stateを更新する場合はactで囲む', () => {
  const { result } = renderHook(() => useCounter())
  expect(result.current.count).toBe(0)

  act(() => {
    result.current.increment()
  })

  expect(result.current.count).toBe(1)
})
```

result.currentを変数に代入できるが、その時点の値から更新されないため注意。
```tsx
  const { result } = renderHook(() => useCounter())

  const {count: prevCount} = result.current
  act(() => {
    result.current.increment()
  })

  const {count: currCount} = result.current

  expect(prevCount).toBe(0)
  expect(currCount).toBe(1)
```

### 非同期にstateを更新する
カスタムフックのuseEffectがstateを更新する場合、renderHookをact関数で囲む。
```tsx
const useCounter = (initialCount: number) => {
  const [count, setCount] = useState(initialCount)
  useEffect(() => {
    setCount(initialCount)
  }, [initialCount])
  const increment = useCallback(() => setCount((x) => x + 1), [])
  return { count, increment }
}

test('非同期でstateを更新する場合もactで囲む', async () => {
  await act(async () => {
    const { result, waitForNextUpdate, rerender } = renderHook((initialCount: number) => useCounter(initialCount), {
      initialProps: 0,
    })
    expect(result.current.count).toBe(0)

    rerender(3)
    await waitForNextUpdate()

    expect(result.current.count).toBe(3)
  })
})
```

### 非同期関数をモック化してテストする
```tsx
describe('非同期関数を使うフックのテスト', () => {
  let mockFetch: jest.Mock
  beforeEach(() => {
    mockFetch = jest.fn((url: string) =>
      Promise.resolve({
        data: url,
      })
    )
    global.fetch = mockFetch
  })

  const useFetch = (props: Params) => {
    const { id } = props
    const [data, setData] = useState<Response | null>(null)

    const fetchData = useCallback(async () => {
      const fetchedData = await fetch(`https://example.com/${id}`)
      setData(fetchedData)
    }, [id])

    useEffect(() => {
      void fetchData()
      return () => {
        setData(null)
      }
    }, [fetchData])

    return { data }
  }

  test('非同期でstateを更新する場合もactで囲む', async () => {
    await act(async () => {
      const { waitForNextUpdate, rerender } = renderHook((params) => useFetch(params), {
        initialProps: {
          id: '1',
        },
      })

      // useEffect内でstateが更新されるまで待つ
      await waitForNextUpdate()

      // propsを変えて再レンダリングすると再度stateが更新される
      rerender({ id: '2' })
      await waitForNextUpdate()
    })

    expect(mockFetch).toBeCalledTimes(2)
  })
})

```

### カスタムフックが例外をスローするのをテストする
```tsx
const { result } = renderHook(useCounter);
expect(result.error).toBeUndefined();
act(() => result.current.incrementByAmountValue(20));
expect(result.error.message).toBe("10までしか加算できません");
```

### wrapper（useContextのテストに有効）
準備中。