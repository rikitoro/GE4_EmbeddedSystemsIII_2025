# 6章 レジスタを用いた応用回路の設計

ここでは、レジスタを用いた応用回路の設計を行います。
複数のレジスタからなる回路を共通のクロック信号で同期して動作させることでより複雑な回路を設計することができます。
実習ボード DE0-CV では、50MHz のクロック信号が供給されているので、このクロック信号を利用して、周期的な動作をもつ回路を設計することができます。

## 6.1  クロックの立ち下がりで値をセットするレジスタ

5 章ではクロックの立ち上がりのタイミングで値をセットするレジスタ回路を設計しました。
ここでは、後での応用を考えて、クロックの**立ち下がり**のタイミングで値をセットするレジスタを設計します。

![register_r_en_n](./assets/register_r_en_n.svg)

<図 6.1 register_r_en_n モジュール>

以下のリスト 6.1 は、クロックの立ち下がりでレジスタに値をセットする register_r_en_n モジュールです。
5.1 で設計した register_r_en モジュールと同様に、リセットと書き込み許可機能を持っています。
パラメータ `WIDTH` でレジスタのビット幅を指定でき、パラメータ `RESET_VALUE` でリセット時の値を指定できます。

< リスト 6.1 書き込み許可付きレジスタ register_r_en_n (クロックの立ち下がりで値をセット) >

```sv : register_r_en_n.sv
module register_r_en_n #(
  parameter WIDTH = 4,
  parameter logic [WIDTH-1:0] RESET_VALUE = '0
)(
  input   logic             clock,
  input   logic             reset,
  input   logic             en,
  input   logic [WIDTH-1:0] d,
  output  logic [WIDTH-1:0] q
);
  
  always_ff @ (negedge clock) begin // クロックの立ち下がりで起動
    if (reset == 1'b1) begin 
      q <= RESET_VALUE;
    end else if (en == 1'b1) begin
      q <= d;
    end 
  end

endmodule
```

この register_r_en_n モジュールは、クロック信号 `clock` の立ち下がりエッジで起動します。
書き込み許可信号 `en` が 1 のときに、入力信号 `d` の値が出力信号 `q` にセットされラッチされます。
リセット信号 `reset` が 1 のときは、出力 `q` にリセット値 `RESET_VALUE` がセットされます。

図 6.1a に register_r_en_n モジュールの動作例をタイムチャートで示します。
パラメータ `WIDTH` を 4 とし、リセット値 `RESET_VALUE` を 4'b0000 とした場合の動作例です。

![register_r_en_n モジュールの動作例](./assets/chap05_register_r_en_n_timing.svg)

<図 6.1a register_r_en_n モジュールの動作例>

### 演習

リスト 6.1 の register_r_en_n モジュール搭載した以下の shell モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  input   logic       SW8,    // write-enable
  input   logic [3:0] SW,     // d (SW3-SW0)
  output  logic [3:0] LEDR    // q (LEDR3-LEDR0)
);

  register_r_en_n reg4_n( // WIDTH = 4, RESET_VALUE = 4'b0000
    .clock (KEY0),
    .reset (SW9),
    .en    (SW8),
    .d     (SW),
    .q     (LEDR)
  );
  
endmodule
```

shell モジュールをプロジェクトの Top level design entity として設定し、その入出力を表 6.1 に示すようにデバイスへ割り当ててください。

<表 6.1 shell モジュールの入出力のデバイスへの割り当て>

| 信号名    |割り当てデバイス| 入出力 |
|-----------|----------------|--------|
| KEY0      | KEY0           | input  |
| SW9       | SW9            | input  |
| SW8       | SW8            | input  |
| SW        | SW3-SW0        | input  |
| LEDR      | LEDR3-LEDR0    | output |


クロックの立ち上がりでレジスタに値をセットする register_r_en モジュールとの動作の違いを確認しましょう。

---

## 6.2  N 進カウンタ (クロックの立ち下がりをカウント)

6.1 で設計した register_r_en_n モジュールを利用して、リセット付きの N 進カウンタを設計します。

![counterN_n](./assets/counterN_n.svg)

<図 6.2 counterN_n モジュール>

次のリスト 6.2 は、クロックの立ち下がりでカウントアップするリセット付き N 進カウンタ counterN_n モジュールです。
パラメータ `WIDTH` でカウンタのビット幅を指定でき、パラメータ `MAX` でカウントの最大値を指定できます。
これにより、0 から `MAX` までをカウントする N 進カウンタ (N = MAX + 1) が実現できます。

<リスト 6.2 N 進カウンタ counterN_n (クロックの立ち下がりでカウントアップ)>

```sv : counterN_n.sv
module counterN_n #(
  parameter WIDTH = 4,
  parameter logic [WIDTH-1:0] MAX = '1
)(
  input   logic             clock,
  input   logic             reset,
  output  logic [WIDTH-1:0] count
);
  
  logic [WIDTH-1:0] next_count;
  
  assign  next_count = (count == MAX) ? '0 : count + 1'd1;
  
  register_r_en_n#(.WIDTH(WIDTH)) reg_count(
    .clock    (clock),
    .reset    (reset),
    .en       (1'b1),
    .d        (next_count),
    .q        (count)
  );

endmodule
```

### 演習

リスト 6.2 の counterN_n モジュールを搭載した以下の shell モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  output  logic [3:0] LEDR0   // count
);
  
  // 10 進カウンタ
  counterN_n#(.MAX(4'd9)) counter10( // WIDTH = 4, 
    .clock  (KEY0),
    .reset  (SW9),
    .count  (LEDR0)
  );


endmodule
```

shell モジュールをプロジェクトの Top level design entity として設定し、その入出力を表 6.2 に示すようにデバイスへ割り当ててください。
プロジェクトには以下のモジュールを定義したファイルが必要となります。
- shell (上記の shell モジュール)
- counterN_n (リスト 6.2 の N 進カウンタ)
- register_r_en_n (リスト 6.1 のレジスタ)


<表 6.2 shell モジュールの入出力のデバイスへの割り当て>

| 信号名    |割り当てデバイス| 入出力 |
|-----------|----------------|--------|
| KEY0      | KEY0           | input  |
| SW9       | SW9            | input  |
| LEDR0     | LEDR3-LEDR0    | output |




---

## 6.3.  N 分周パルス発生器

N 進カウンタを利用して、クロック信号を N 分周するパルス発生器を設計します。
図 6.3 に N 分周パルス発生器 pulse_dividerN モジュールを示します。
内部の N 進カウンタ counterN_n を利用して (N = MAX + 1)、クロック `clock` の立ち下がりでカウントアップし、カウント値が `MAX` に達したときにちょうど 1 クロック分のパルス信号を `pulse` から出力します。
これにより、クロック信号を `MAX + 1` 分周したパルス信号を出力します。

![pulse_dividerN](./assets/pulse_dividerN.svg)

<図 6.3 pulse_dividerN モジュール>

次のリスト 6.3 は、N 分周パルス発生器 pulse_dividerN モジュールです。
パラメータ `WIDTH` でカウンタのビット幅を指定し、パラメータ `MAX` でカウントの最大値を指定します。

<リスト 6.3 N 分周パルス発生器 pulse_dividerN>

```sv : pulse_dividerN.sv
module pulse_dividerN #(
  parameter WIDTH = 4,
  parameter logic [WIDTH-1:0] MAX
)(
  input   logic clock,  // クロック信号 (立ち下がりでカウントアップ)
  input   logic reset,
  output  logic pulse   // パルス信号 
);

  logic [WIDTH-1:0] count;  // カウンタのカウント値

  assign pulse = (count == MAX) ? 1'b1 : 1'b0; // カウントが MAX のときにパルスを出力
  
  counterN_n #(.WIDTH(WIDTH), .MAX(MAX)) counter( // N 進カウンタ (N = MAX + 1)
    .clock  (clock),
    .reset  (reset),
    .count  (count)
  );
  
endmodule
```

この pulse_dividerN モジュールの動作例を図 6.3a に示します。
この例では、パラメータ `WIDTH` を 2、`MAX` を 2 として、3 分周パルス (N = MAX + 1 = 3) を生成しています。
クロックの立ち下がりエッジでカウント値 `count` がカウントアップし、その値が 2 (MAX) に達したときにパルス信号 `pulse` が 1 となります。
なお、カウント値 `count` は、モジュール内部の信号であり外部からは観測できない信号ですので、タイムチャートでは破線で示しています。
出力信号 `pulse` は、3 クロックサイクルごとに 1 クロック周期分のパルスを出力します。

![pulse_dividerN_n モジュールの動作例](./assets/pulse_dividerN_timing.svg)

<図 6.3a pulse_dividerN_n モジュールの動作例>

### 演習

リスト 6.3 の pulse_dividerN_n モジュールを搭載した以下の shell モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。

```sv : shell.sv
module shell(
  input   logic     KEY0, // clock
  input   logic     SW9,  // reset
  output  logic     LEDR0 // pulse 
);

  pulse_dividerN #(.WIDTH(4), .MAX(4'd9)) // 10 分周パルス発生器 
  pulse_div10(
    .clock  (KEY0),
    .reset  (SW9),
    .pulse  (LEDR0)
  );

endmodule
```

---

## 6.4  1 秒ごとにカウントアップするカウンタ

実習ボード DE0-CV には 50MHz のクロックが供給されています。
このクロックを利用して、1 秒ごとにカウントアップする 10 進カウンタを設計します。

6.3 で設計した N 分周パルス発生器を用いて、50MHz クロックを 50,000,000 分周するパルス信号を生成すると、1 秒ごとに 1 クロック分のパルス信号が出力されます。
このパルス信号をカウント許可付きカウンタのカウント許可信号として利用し、1 秒ごとにカウントアップする 10 進カウンタを実現します。

図 6.4 に回路の構成を構成を示します。
5000 万分周パルス発生器 pulse_dividerN_n (`WIDTH` = 26, `MAX` = 49_999_999) と、
10 進カウンタ counterN_en (`WIDTH` = 4, `MAX` = 9) を組み合わせて、1 秒ごとにカウントアップする 10 進カウンタを実現しています。

![counter10_onesec](./assets/counter10_onesec.svg)

<図 6.4 1 秒ごとにカウントアップする 10 進カウンタ>

図 6.4a 回路の動作例をタイムチャートで示します。
5000 万分周パルス発生器 pulse_dividerN_n は、50MHz クロックの立ち下りを 50,000,000 回カウントするたびに、クロック 1 周期分のパルス信号 `onesec_pulse` を出力します。
すなわち、1 秒ごとに 1 クロック分のパルス信号を出力します。
10 進カウンタ counterN_en は、この `onesec_pulse` 信号をカウント許可信号として利用して、50 MHz クロックの上がりカウントアップします。
なお、カウント許可信号として用いる `onesec_pulse` 信号が 1 となっている状態で安定的にクロック信号の立ち上がりが来るように、パルス発生器 pulse_dividerN_n はクロックの立ち下がりで起動し、10 進カウンタ counterN_en はクロックの立ち上がりで起動するようにしています。

![counter10_onesec_timing](./assets/counter10_onesec_timing.svg)

<図 6.4a 回路の動作例>

次のリスト 6.4 は、1 秒ごとにカウントアップする 10 進カウンタを実現する shell モジュールです。

```sv : shell.sv
module shell(
  input   logic       CLOCK_50, // clock (50MHz)
  input   logic       SW9,      // reset
  output  logic [3:0] LEDR     // count
);

  logic onesec_pulse;  // 1 秒ごとのパルス信号
  
  // 1 秒ごとのパルス発生器 (50MHz クロックを 50,000,000 分周)
  pulse_dividerN_n#(.WIDTH(26), .MAX(26'd49_999_999)) onesec_gen(
    .clock  (CLOCK_50),
    .reset  (SW9),
    .pulse  (onesec_pulse)
  );
  
  // 10 進カウンタ (1 秒ごとにカウントアップ)
  counterN_en #(.WIDTH(4), .MAX(4'd9)) counter10(
    .clock  (CLOCK_50),
    .reset  (SW9),
    .en     (onesec_pulse),
    .count  (LEDR)
  );
  
endmodule
```

### 演習

リスト 6.4 の shell モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。
DE0-CV のでは、CLOCK_50 信号が 50MHz のクロック信号として供給されています。
shell モジュールをプロジェクトの Top level design entity として設定し、その入出力を表 6.3 に示すようにデバイスへ割り当ててください。

<表 6.4 shell モジュールの入出力のデバイスへの割り当て>

| 信号名    |割り当てデバイス| 入出力 |
|-----------|----------------|--------|
| CLOCK_50  | CLOCK_50       | input  |
| SW9       | SW9            | input  |
| LEDR      | LEDR3-LEDR0    | output |


---

## 6.5  課題 (100 Hz のパルス信号発生器)

実習ボード DE0-CV では、50MHz のクロック信号が供給されています。
このクロック信号を利用して、100 Hz のパルス信号を発生させるパルス発生器を設計しましょう。

発生させるパルス信号は図 6.5 に示すような 100 Hz の矩形波です。
1 周期 \\(T\\) のうち、パルスが 1 となる時間を \\(T_{on} = 0.3 T\\) と設定してください。

![pulse_100Hz](./assets/chap06_ex_pulse.svg)

<図 6.5 100 Hz のパルス波形>

発生したパルス信号は、実習ボードの GPIO Header のいずれかのピン (例えば GPIO_1_D1) に出力し、オシロスコープを用いて出力波形を確認してください。


