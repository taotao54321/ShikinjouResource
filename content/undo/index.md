+++
title = "undo 関連の不具合"
date = 2023-05-10
+++

このゲームは A キーを押すことで着手の undo ができる(最大 5 手まで)。  
しかし、この undo 機能にはいくつかの不具合がある。

本ページ内の動画では [Lua スクリプト](https://gist.github.com/taotao54321/96c2b1b329613cf67d0282e16ef2d73a)を用いて牌や落とし穴を可視化している。


## 牌以外のオブジェクトが復元されない

牌が落とし穴の上を通過すると落とし穴は消えるが、undo しても消えたまま:

{{ video(src="pit.mp4") }}

つるはしで牌を消す手は直前の手とマージして undo されるが、つるはし自体は復活しない:

{{ video(src="pickaxe.mp4") }}

なお、テレポートに関してはこの問題はない。テレポートを行った時点で undo 履歴はクリアされるため:

{{ video(src="teleport.mp4") }}


## redo バグ

着手を行った後、外壁にぶつかりながら undo を行うと、何も入力しなくても直前の着手が redo される:

{{ video(src="redo.mp4") }}

この redo の間、自機は自由に動ける。  
このとき redo 完了前に「ルール上押せない牌」を押すと、「押せない原因となった牌」が redo される着手と同じ方向に動く。  
この動作は障害物を無視する。また、画面表示が正しく更新されない。

たとえば、以下では押せない原因の牌は自機の右隣の西(お邪魔牌)であり、redo 手が下方向なので、この西が下へ動く:

{{ video(src="redo-bug-1.mp4") }}

一方、押せない牌 A がお邪魔牌でなく、かつ 1 歩先に牌 B があるなら、牌 B が押せない原因の牌として扱われる。  
たとえば、以下では押せない原因の牌は自機の 2 歩右の一索であり、redo 手が下方向なので、この一索が下へ動く:

{{ video(src="redo-bug-2.mp4") }}

このバグにより、同じマスに複数の牌が存在しうる。その場合、牌リスト内でインデックスが最小の牌が優先して処理される。

ここまでの例では自機に隣接した押せない牌を使って redo バグを発動したが、redo 手が縦方向の場合、1 歩横に歩いてからバグを発動することもできる。  
これは横方向と縦方向で移動にかかる時間が違う(8F 差がある)のが理由。  
たとえば、以下では 1 歩左に歩いてから左の六筒に触れた結果、押せない原因である 2 歩左のお邪魔牌が上へ動いている:

{{ video(src="redo-bug-3.mp4") }}

これを利用すると、上下の壁際の牌を盤面外へ押し出すこともできる。  
たとえば、以下では左上隅のゴール扉が上へ動き、盤面外へ消えている:

{{ video(src="redo-bug-4.mp4") }}

盤面外へ消えた牌はおそらく盤上に影響することはないと思われる。  
また、これを利用したメモリ破壊手段は今のところ見つかっていない。


## 牌バッファオーバーフロー

undo 履歴のエントリは着手により消えた牌たちを記録しているが、この牌バッファは最大 4 個までの牌しか記録できない。  
紫禁城のルールでは 1 手で 5 個以上の牌が同時に消えることはないので、通常はこれで問題ないが、つるはしのある面ではオーバーフローが発生しうる。  
(つるはしによって消えた牌は直前の履歴エントリの牌バッファに追加されるため)

このバッファオーバーフローが生じた後、さらに 1 手着手してから 2 手 undo すると、undo 処理が壊れて盤面がメチャクチャになる。  
143 面での実行例を示す:

{{ video(src="overflow.mp4") }}

このバグは undo 履歴と牌リストを破壊するが、他のメモリ領域はおそらく破壊されないので、任意コード実行などには繋がらないと思われる。
