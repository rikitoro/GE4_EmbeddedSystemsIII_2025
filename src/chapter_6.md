# 6章 レジスタを用いた応用回路の設計



## 6.1 クロックの立ち下がりで値をセットするレジスタ

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
| KEY0      | KEY0           | 入力   |
| SW9       | SW9            | 入力   |
| SW8       | SW8            | 入力   |
| SW        | SW3-SW0        | 入力   |
| LEDR      | LEDR3-LEDR0    | 出力   |


クロックの立ち上がりでレジスタに値をセットする register_r_en モジュールとの動作の違いを確認しましょう。

---

## 6.2 N 進カウンタ (クロックの立ち下がりをカウント)

6.1 で設計した register_r_en_n モジュールを利用して、リセット付きの N 進カウンタを設計します。

![counterN_n](./assets/counterN_n.svg)

<図 6.2 counterN_n モジュール>

次のリスト 6.2 は、クロックの立ち下がりでカウントアップするリセット付き N 進カウンタ counterN_n モジュールです。
パラメータ `WIDTH` でカウンタのビット幅を指定でき、パラメータ `MAX` でカウントの最大値を指定できます。

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
---

## 6.3 パルス発生器 (N 進カウンタを利用)

```sv : pulse_dividerN_n.sv
module pulse_dividerN_n #(
  parameter WIDTH = 4,
  parameter logic [WIDTH-1:0] MAX
)(
  input   logic clock,
  input   logic reset,
  output  logic pulse
);

  logic [WIDTH-1:0] count;
  
  assign pulse = (count == MAX) ? 1'b1 : 1'b0;
  
  counterN_n #(.WIDTH(WIDTH), .MAX(MAX)) counter(
    .clock  (clock),
    .reset  (reset),
    .count  (count)
  );
  

endmodule
```

```sv : shell.sv
module shell(
  input   logic     KEY0, // clock
  input   logic     SW9,  // reset
  output  logic     LEDR0 // pulse 
);

  pulse_dividerN_n #(.WIDTH(4), .MAX(4'd9)) 
  pulse_div10(
    .clock  (KEY0),
    .reset  (SW9),
    .pulse  (LEDR0)
  );


endmodule
```

---

## 6.4 

```sv : shell.sv
module shell(
  input   logic       CLOCK_50, // clock (50MHz)
  input   logic       SW9,      // reset
  output  logic [3:0] LEDR,     // count
  output  logic [6:0] HEX0
);

  logic onesec_pulse;
  
  pulse_dividerN_n#(.WIDTH(26), .MAX(26'd49_999_999)) onesec_gen(
    .clock  (CLOCK_50),
    .reset  (SW9),
    .pulse  (onesec_pulse)
  );
  
  counterN_en #(.WIDTH(4), .MAX(4'd9)) counter10(
    .clock  (CLOCK_50),
    .reset  (SW9),
    .en     (onesec_pulse),
    .count  (LEDR)
  );
  
  sseg_decoder dec(
    .num  (LEDR),
    .hex  (HEX0)
  );
  

endmodule
```