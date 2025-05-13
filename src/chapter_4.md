# 4 章 構造モデル化による回路設計

System Verilog ではモジュールを組み合わせ、より大規模な回路モジュールを構築することができます。
本章では、これまでに設計した様々な回路モジュールを組み合わせて新しい回路を設計する方法を学びます。

## 4.1 7 セグメント表示付きプライオリティエンコーダ

3 章ではプライオリティエンコーダ (priori_encoder) や 7 セグメントデコーダ (sseg_decoder) を作成しました。
ここでは、プライオリティエンコーダへの入力と出力とをそれぞれ 7 セグメントデコーダを用いて変換し、16進数表示のパターンとして 7 セグメント LED 上で表示する回路を設計します。

図 4.1a のように、priori_encoder を 1 個と sseg_decoder を 2 個組み合わせてより大きな回路モジュール shell を設計していきます。

![7セグメント表示付きプライオリティエンコーダ](./assets/chap04_penc_hex.svg)

<図 4.1a 7セグメント表示付きプライオリティエンコーダ>

外側の shell モジュールは、入力として 4 ビットの信号 SW があり、
出力としては、1 ビットの信号 LEDR と 7 ビットの信号 HEX1, HEX0 があります。
ここでは、これらの入出力信号について、入力の SW は実習ボードのスライドスイッチへ接続し、
出力の LEDR は実習ボードの LED へ、HEX1 と HEX0 は 7 セグメント LED ディスプレイへ接続することを想定しています。

なお、Quartus Prime では回路を構成するモジュールのうち一番外側で FPGA の入出力に直接つながる回路モジュールの名前を
プロジェクトの Top level design entity の名前にする必要があります。

それでは、Quartus Prime で Top level design entity を shell と設定したプロジェクトを作成しましょう。

プロジェクトを作成したら、リスト 4.1a に示した shell モジュールを定義したファイルをプロジェクトに登録してください。
なお、ここでは shell モジュールの入出力の定義だけを記述しています。モジュールの内部の記述は後程行います。

<リスト4.1a shell モジュール (作成途中版) >
```sv : shell.sv
module shell(
  input   logic [3:0] SW,
  output  logic [6:0] HEX0,
  output  logic [6:0] HEX1,
  output  logic       LEDR0
);
  // 回路の中身は後で作成 
endmodule
```

shell モジュールの内部では、priority_encoder モジュールと sseg_decoder モジュールを呼び出して使用します。
したがって、これらのモジュールを定義したファイルも必要となりますので、プロジェクトに追加して登録してください。
(プロジェクトには shell.sv, priotiry_encode.sv, sseg_decoder.sv のファイルが登録されていることを確認してください)

<リスト4.1b priority_encoder モジュール (再掲) >
~~~admonish title="priority_encoder.sv", collapsible=true
```sv : priority_encoder.sv
module priority_encoder(
  input   logic [3:0] d,
  output  logic [1:0] y,
  output  logic       en
);

  assign en = |d; 
  
  always_comb begin
    casez (d)
      4'b1??? : y = 2'b11;
      4'b01?? : y = 2'b10;
      4'b001? : y = 2'b01;
      4'b0001 : y = 2'b00;
      default : y = 2'b00;
     endcase
  end
  
endmodule
```
~~~

<リスト4.1c sseg_decoder モジュール (再掲) >
~~~admonish title="sseg_decoder.sv", collapsible=true
```sv : sseg_decoder.sv
module sseg_decoder(
  input   logic [3:0]   num,
  output  logic [6:0]   hex
);

  always_comb begin
    case (num)
      4'h0  : hex = 7'b100_0000;
      4'h1  : hex = 7'b111_1001;
      4'h2  : hex = 7'b010_0100;
      4'h3  : hex = 7'b011_0000;
      4'h4  : hex = 7'b001_1001;
      4'h5  : hex = 7'b001_0010;
      4'h6  : hex = 7'b000_0010;
      4'h7  : hex = 7'b101_1000;
      4'h8  : hex = 7'b000_0000;
      4'h9  : hex = 7'b001_0000;
      4'ha  : hex = 7'b000_1000;
      4'hb  : hex = 7'b000_0011;
      4'hc  : hex = 7'b100_0110;
      4'hd  : hex = 7'b010_0001;
      4'he  : hex = 7'b000_0110;
      4'hf  : hex = 7'b000_1110;
      default  : hex = 7'b111_1111; 
    endcase
  end

endmodule
```
~~~

図 4.1a をもとにすると shell モジュールの内部はリスト 4.1d のように記述することができます。

<リスト4.1d shell モジュール (完成版) >
```sv : shell.sv
module shell(
  input   logic [3:0] SW,
  output  logic [6:0] HEX0,
  output  logic [6:0] HEX1,
  output  logic       LEDR0
);

  logic [1:0] encoded;
  
  priority_encoder p_enc(
    .d  (SW),
    .y  (encoded),
    .en (LEDR0)
  );
  
  sseg_decoder dec1(
    .num  ({2'd00, encoded}),
    .hex  (HEX1)
  );
  
  sseg_decoder dec0(
    .num  (SW),
    .hex  (HEX0)
  );

endmodule
```




### 演習

セグメント表示付きプライオリティエンコーダを実習ボード DE0-CV に実装してその動作を確認しましょう。
shell モジュールを top level モジュールとして
その入出力の信号を実習ボードのデバイスへ表 4.1 のように割り当ててください。

なお、Quartus Prime のプロジェクトを作成する際は、トップレベルエンティティとして shell モジュールを指定してください
また、shell モジュールに加え、priority_encoder モジュールと sseg_decoder モジュールを定義しているファイルもプロジェクトに登録しておく必要があります。注意してください。

<表4.1 shell モジュールの入出力のデバイスへの割り当て>

| 信号名   |割り当てデバイス| 入出力 |
|----------|----------------|--------|
|SW[3:0]   | SW3-SW0        | input  |
|HEX0[6:0] | HEX06-HEX00    | output |
|HEX1[6:0] | HEX16-HEX10    | output |
|LEDR0     | LEDR0          | output |


## 4.2 課題 (入出力表示付き 4 ビット加算器)

4ビット加算器 adder モジュールと 7 セグメントデコーダ sseg_decoder モジュールを組み合わせて、
以下に仕様を示すような、入出力表示付き 4 ビット加算器を設計し、ボードに実装してその動作を確認してください。

### 回路の仕様

入力部:

| 使用デバイス | 説明 |
|--------------|------|
| SW7-SW4      | 4 ビットの加算器の入力 A |
| SW3-SW0      | 4 ビットの加算器の入力 B |

出力部:

| 使用デバイス | 説明 |
|--------------|------|
| HEX3, HEX2   | 入力AとBの和を2桁の16進数で表示 |
| HEX1         | 入力Aの値を16進数で表示 |
| HEX0         | 入力Bの値を16進数で表示 |

回路の動作: 

- スライドスイッチ SW7-SW4 より 4 ビットの値 A を入力し、SW3-SW0 より 4 ビットの値 B を入力します。
- 7セグメント LED の HEX1 には A の値を、HEX0 には B の値をそれぞれ 16 進数のパターンを表示します。
- 7セグメント LED HEX3 と HEX2 には A + B の結果を 16 進数 2 桁で表示します。


### 回路の動作例

- A = 0001, B = 1010 を入力した場合 : 
  - HEX3, HEX2 : 0 b のパターンを表示
  - HEX1 : 1 のパターンを表示 
  - HEX0 : A のパターンを表示 

- A = 1111, B = 0011 を入力した場合 :
  - HEX3, HEX2 : 1 2 のパターンを表示
  - HEX1 : F のパターンを表示
  - HEX0 : 3 のパターンを表示
