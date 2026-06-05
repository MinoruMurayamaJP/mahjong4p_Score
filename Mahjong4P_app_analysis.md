# Mahjong4P_Cal_T.xlsx 解析メモ

## 結論

iPhoneアプリ化できます。

このExcelは複雑な麻雀役・符・翻の点数計算表ではなく、4人麻雀のスコア収支を局/回ごとに入力し、各プレイヤーの合計と換算額を出すスコア記録表です。SwiftUIでかなり素直に再現できます。

## Excelの構造

- シート: `麻雀スコア`
- 主な使用範囲: `A1:G43`
- プレイヤー欄: `B1:E1`
- 予備プレイヤー欄らしき列: `F1`、現在は空欄
- 換算倍率: `A1`、現在値は `50`
- 合計行: `B2:F2`
- 換算後金額行: `B3:F3`
- 入力履歴: 1から20回分、各回2行構成

## 読み取れた計算ルール

各プレイヤーの合計:

```text
プレイヤー合計 = そのプレイヤー列の4行目から43行目までの合計
```

Excel式の例:

```excel
=IF(B$1<>"",SUM(B4:B43),"")
```

換算後金額:

```text
換算後金額 = A1の倍率 × プレイヤー合計
```

Excel式の例:

```excel
=IF(B$1<>"",$A$1*B2,"")
```

各回の入力補完:

```text
G列 = その回の入力行 B:F の合計
下段セル = 上段セルが空欄で、プレイヤー名がある場合、-G列を入れる
```

Excel式の例:

```excel
=SUM(B4:F4)
=IF(B4="",IF(B$1<>"",-$G4,""),"")
```

つまり、ある回で一部プレイヤーの収支を上段に入力すると、未入力のプレイヤー側に差額を自動補完する設計です。

## iPhoneアプリ案

最小構成:

- プレイヤー4人の名前入力
- 換算倍率入力
- 20回分のスコア入力
- 各回の差額自動補完
- 合計スコア表示
- 換算後金額表示
- リセット/保存

推奨構成:

- SwiftUI
- データ保存: `UserDefaults` または `SwiftData`
- 計算ロジック: Excel数式ではなくSwiftの純粋関数に置き換え
- 画面: 上部に合計、下部に回ごとの入力リスト

## Swift側の計算モデル例

```swift
struct Player: Identifiable {
    let id = UUID()
    var name: String
}

struct RoundScore: Identifiable {
    let id = UUID()
    var inputs: [Int?] // 4人分。入力なしは nil
}

func completedScores(for round: RoundScore, playerCount: Int) -> [Int?] {
    let sum = round.inputs.compactMap { $0 }.reduce(0, +)
    return round.inputs.enumerated().map { index, value in
        if value != nil {
            return value
        }
        return -sum
    }
}

func totals(rounds: [RoundScore], playerCount: Int) -> [Int] {
    var result = Array(repeating: 0, count: playerCount)
    for round in rounds {
        let scores = completedScores(for: round, playerCount: playerCount)
        for index in 0..<playerCount {
            result[index] += scores[index] ?? 0
        }
    }
    return result
}

func convertedTotals(totals: [Int], rate: Int) -> [Int] {
    totals.map { $0 * rate }
}
```

注意: Excelの式をそのまま再現すると、1回の上段で複数プレイヤーが空欄の場合、それぞれに同じ差額が入ります。アプリでは「差額補完するプレイヤーを1人選ぶ」UIにした方が誤入力を防げます。

## 次に作るなら

1. SwiftUIのiPhone画面を作る
2. Excelのサンプル値と同じ結果になる単体テストを作る
3. 保存機能を入れる
4. 必要ならExcel出力/共有機能を追加する
