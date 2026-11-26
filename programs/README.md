# programs のビルド環境

各プログラムを書き込む先のボードと、当時使っていたと思われる ESP32 コア・ライブラリの版をまとめる。
2021 年当時の正確な版はどこにも残っていないため、**「推定」と書いた版はコードとコミット時期からの推定**で、確かめていない。

## プログラムと書き込み先

| フォルダ | 書き込み先 | ボードの指定 |
| --- | --- | --- |
| `M5swarm_Display/` | M5Stack（ESP32）。Wi-Fi で HTTP を受けて足回り基板へ SPI で指令を送る | 当時: `espressif:esp32:m5stack-core-esp32`（手で置いた arduino-esp32）<br>今のボードマネージャ版で同じボード: `esp32:esp32:m5stack-core-esp32`（2.0 系まで）/ `esp32:esp32:m5stack_core`（3.0 系から） |
| `M5swarm_Blynk/` | M5Stack（ESP32）。Blynk の Bluetooth 接続で操作する | 同上 |
| `M5swarm_bottom/` | 足回り基板（ATmega328P-AU） | `arduino:avr:pro`（Arduino Pro or Pro Mini）。CPU の選び方は下の「足回り基板の書き込み設定」を参照 |

M5Stack 用2本の `.vscode/arduino.json` に残っているボード設定:
`FlashMode=qio, FlashFreq=80, PartitionScheme=default, UploadSpeed=921600, DebugLevel=none`

Python の補助スクリプト（`M5swarm_Display/a.py`、`swarm_robotics.py`）は、共通で `opencv-python`、`numpy`、`mediapipe` を使う。ほかに `swarm_robotics.py` は `requests`、`a.py` は `matplotlib` を使う。版は分からない。

## 当時の版

| もの | 版 | 確かさ | 根拠 |
| --- | --- | --- | --- |
| Arduino IDE | 1.8.13 | 記録あり | `.vscode/c_cpp_properties.json` の `ARDUINO=10813` |
| ESP32 コア（arduino-esp32） | 1.0 系（1.0.6 が有力） | 推定 | `c_cpp_properties.json` のコンパイラが xtensa gcc 5.2.0 で、`tools/sdk/include/...` の構成も 1.0 系のもの（2.0.0 からは gcc 8.4.0）。`hardware/espressif/esp32` に手で置く入れ方。2021-11 時点の 1.0 系の最終版は 1.0.6（2021-03）。git clone で入れていた場合は、リリース版ではなく 1.0 系の時期の master の可能性もある |
| M5Stack ライブラリ | 0.3 系（0.3.9 以下） | 推定 | 根拠は時期だけ。最終コミット（2021-11-30）時点の最新は 0.3.9（2021-11-08）、0.4.0 は 2022-03。使っている API は 0.4 系にもあるので、コードからは決められない |
| Blynk ライブラリ | 0.6.1 か 1.0.x | 推定 | `BlynkSimpleEsp32_BT.h` と `BLYNK_USE_DIRECT_CONNECT` を使い、テンプレート ID なしで認証する旧 Blynk の書き方。1.0.0 は 2021-05 に出ており、どちらかは分からない |
| Arduino AVR Boards（足回り基板） | 1.8.3 | 推定 | Arduino IDE 1.8.13 に同梱の版。ボードマネージャで更新していれば 1.8.4（タグは 2021-11-23）の可能性もある |

推定どおりの環境を arduino-cli で揃える場合の例（未実行・未確認）:

```sh
arduino-cli core install esp32:esp32@1.0.6 --additional-urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
arduino-cli core install arduino:avr@1.8.3
arduino-cli lib install M5Stack@0.3.9 Blynk@0.6.1
```

## 今の環境でのコンパイル結果

2026-11-26 に、手元にあった環境（Arduino IDE 2 同梱の arduino-cli 1.5.1）で試した結果。**当時の版では試していない。**

| フォルダ | 使った環境 | 結果 |
| --- | --- | --- |
| `M5swarm_bottom/` | Arduino AVR Boards 1.8.8、`arduino:avr:pro:cpu=16MHzatmega328` と `cpu=8MHzatmega328` | 両方とも通る（5864 バイト） |
| `M5swarm_Display/` | ESP32 コア 3.3.11、`esp32:esp32:m5stack_core`、M5Stack ライブラリ 0.4.6 | 通らない。M5Stack ライブラリの `pngle.c` が `rom/miniz.h` を見つけられない（3.3.11 では `esp_rom/include/miniz.h` の場所にある）。スケッチ本体まで進まない |
| `M5swarm_Blynk/` | 同上、Blynk ライブラリ 1.3.5 | 同じ理由で通らない。フォルダに `.ino` が2つあり、そのままでは二重定義になる問題（issue #2）は別にある |

M5Stack 用の2つは、ESP32 コア 3.3.11 と M5Stack ライブラリ 0.4.6 の組み合わせでは通らない。上の推定の版（ESP32 コア 1.0 系）か 2.0 系のコアで試す必要がある。足回り基板用も、試したのは 1.8.8 で、当時の版（1.8.3 と推定）では試していない。

## 足回り基板の書き込み設定（要確認）

作者のブログ（URL はこのリポジトリに無い）では「Arduino Pro Mini（5V, 16MHz）」で書き込むとしているが、回路図（`animacySwarmRoboticsPCB/motorBase.sch`）では次のとおりで、合っているか確かめていない。

- マイコン（ATmega328P-AU）の電源は 3.3V（回路図のネット名 `3V3`）。配線は1本ずつは追っていない。
- 水晶（Y2）は「Crystal」とだけあり、周波数の記載が無い。フットプリントは `akizukit:FC-135`。FC-135 は 32.768kHz の時計用水晶によく使われる型だが、実際に載っている部品は確かめていない。

ATmega328P のデータシートでは、上限は 2.7V で 10MHz、4.5V で 20MHz で、その間は直線で変わる。3.3V での上限は約 13MHz で、16MHz は規格外になる。

- 基板の水晶が 16MHz なら、「5V, 16MHz」の指定でタイマーとシリアルの速さは合う（ただし 3.3V で 16MHz は規格外の動作）。
- 水晶が 8MHz なら「3.3V, 8MHz」（`arduino:avr:pro:cpu=8MHzatmega328`）を選ぶ必要がある。16MHz の指定のままだと `millis()`・`delay()`・シリアルの速さが半分になる。

- 水晶が 32.768kHz なら、どちらの指定も合わない（内蔵 8MHz で動かす設定が要る可能性がある）。

実機の水晶の刻印とヒューズの設定で確かめること。
