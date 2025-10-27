---
title: "HP DL160 Gen9のCPU交換作業"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
## はじめに
オークションで購入したDL160だが、6C/6TのCPUだったため、
10C/20TのXeon E5-2640 V4を手に入れて交換してみた。
ヒートシンクを外す際、トルクスドライバーが必要なので注意。

1. このヒートシンクを取り外す。今回はシングル構成なので、もう一つは空きスロット。
![](https://storage.googleapis.com/zenn-user-upload/7608b6ab0cd5-20251027.jpg)

2. ヒートシンクを外すと、グリスがべったりついたCPUが登場。
![](https://storage.googleapis.com/zenn-user-upload/d02596e28987-20251027.jpg)

3. グリスを拭き取っておき、CPUを固定している両側のロックを外すと、CPUがパーツにハマった状態。CPUをパーツから外すが、結構きつくついていた。
![](https://storage.googleapis.com/zenn-user-upload/67e2db811683-20251027.jpg)

4. CPUを付けたパーツをCPUカバーの方へ向きを合わせて入れ、カバー金具で固定する。
![](https://storage.googleapis.com/zenn-user-upload/afe03e72c351-20251027.jpg)
![](https://storage.googleapis.com/zenn-user-upload/026f99706696-20251027.jpg)

5. 最後にグリスをつけて、ヒートシンクをねじ止めして終了。
![](https://storage.googleapis.com/zenn-user-upload/664e71fc5165-20251027.jpg)

6. ESXiとかそのまま入っていて何かエラー吐いたりするかと思ったが、そのまま起動して問題なし。CPUが認識されていることも確認できたが、ハイパースレッディングは無効になっていた。
![](https://storage.googleapis.com/zenn-user-upload/4816ba01515a-20251027.jpg)

## まとめ
- CPUのカバー金具が固いのと、順番があるので注意が必要。
- カバー金具とパーツがあるためにピンを曲げる可能性が低いため、少し気持ちは楽だった。（デスクトップPCとかはめちゃ怖い）
- すんなり認識されたので少し拍子抜け。
- ハイパースレッディングは、BIOSのシステム設定から変更し、問題なく有効化できた。