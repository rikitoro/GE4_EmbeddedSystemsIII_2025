# 6章 ステートマシンの設計

前章までで、レジスタの設計方法と組み合わせ回路の設計方法、さらにそれらの回路モジュールを組み合わせて新しい回路を設計する方法を学んできました。
レジスタと組み合わせ回路とを組み合わせることで、ステートマシン(有限状態機械)を設計することができます。
ステートマシンは内部に状態を持ち、入力に応じて状態を遷移し、出力を生成する回路です。
ステートマシンは、コンピュータのCPUや、通信プロトコル、制御回路など、様々な分野で広く利用されています。
ステートマシンの設計方法を学ぶことで、より複雑な回路を設計できるようになります。


## 6.1 Moore 型ステートマシンの設計

Moore 型ステートマシンは、出力が現在の状態のみに依存するステートマシンです。
状態遷移は入力と現在の状態に基づいて次の状態が決まりますが、出力は現在の状態にのみ依存します。

ここではステートマシンの例として、図 6.1a に示すような 1-bit 入力 p、クロック入力 clock、同期リセット信号 reset、1 bit の出力 y を持つステートマシン my_stm1を設計します。

![ステートマシン my_stm1](./assets/my_stm1.svg "ステートマシン my_stm")

<図6.1a 状態機械 my_stm1 とその状態遷移図>

ステートマシン my_stm1 の状態遷移は図 6.1a の右側に示す状態遷移図で与えます。
my_stm1 は 3 つの状態(SA, SB, SC)を持ち、出力 y が状態だけで決まるような Moore 型の状態機械です。
状態遷移は clock の立ち上がりのタイミングで起こるものとします。
同期リセット信号 reset が 1 になると、clock の立ち上がりで状態は SA にリセットされます。

### 状態の符号化

ステートマシン my_stm1 は 3 つの状態を持ちます。
ステートマシンを実現するには、現在どの状態にあるかを表現し保持するために、取りうる状態を 2 進数のビット列に符号化する必要があります。
符号化にはいろいろな方法がありますが、ここでは one-hot 符号化を用いることにします。

one-hot 符号化では、状態の数だけビットを用意し、現在の状態に対応するビットだけが 1 で他は 0 となるように符号化します。
この場合、3 つの状態(SA, SB, SC)を表すには 3 ビットのレジスタが必要です。
以下の表 6.1a に、状態と符号化されたビット列を示します。

<表6.1a 状態の符号化>

| 状態 | 符号 |
|------|------|
| SA   | 100  |
| SB   | 010  |
| SC   | 001  |

### 状態レジスタの設計

状態マシン my_stm1 の現在の状態を保持するために、3 ビットのレジスタを用意します。
ここでは、このレジスタを状態レジスタと呼ぶことにします。

状態レジスタは 5.3 節で学んだ同期リセット付きレジスタを用いて設計できます。
同期リセット付きレジスタ register_r モジュールの HDL 記述を再掲します。

~~~admonish title="register_r.sv", collapsible=true
```sv : register_r.sv
module register_r #(
  parameter WIDTH = 4,  // 入出力のビット幅 (デフォルトは 4)
  parameter logic [WIDTH-1:0] RESET_VALUE = '0 // リセット時の値 (デフォルトは 0)  
)(
  input   logic             clock,
  input   logic             reset, // reset
  input   logic [WIDTH-1:0] d,
  output  logic [WIDTH-1:0] q
);

  always_ff @ (posedge clock) begin
    if (reset == 1'b1) begin
      q <= RESET_VALUE; // reset が 1 のときは q にリセット値を設定
    end else begin 
      q <= d;           // reset が 0 のときは q に d の値を設定
    end 
  end

endmodule
```
~~~

ステートマシン my_stm1 の状態は 3 ビットで表現され、リセット時には状態 SA にリセットされるので、次のように register_r モジュールをインスタンス化することで状態レジスタを設計できます。

```sv : 
  register_r #(
    .WIDTH(3),            // 3 ビットのレジスタ 
    .RESET_VALUE(3'b100)  // SA にリセット
  ) state_register (
    .clock(clock),
    .reset(reset),
    .d(next_state),
    .q(state)
  );
```

### 次状態関数回路

次状態関数回路は、現在の状態と入力に基づいて次の状態を決定する回路です。
図 6.1a の状態遷移図をもとに、現在の状態と入力 p の組み合わせから次の状態を決定するための表を作成します。

| 今の状態 | 入力 p | 次の状態 |
|----------|--------|----------|
| SA       | 0      | SA       |
| SA       | 1      | SB       |
| SB       | 0      | SC       |
| SB       | 1      | SA       |
| SC       | 0      | SC       |
| SC       | 1      | SA       |




表6.1b の状態遷移表に現れる状態を、表6.1a によって符号に置き換えることで、次状態関数回路 next_state_generator モジュールの真理値表が得られます。
<表6.2 my_stm の状態遷移表>

| p | 今の状態 | 次の状態 |
|---|----------|----------|
| 0 | SA | SB |
| 0 | SB | SC |
| 0 | SC | SD |
| 0 | SD | SA |
| 1 | SA | SA |
| 1 | SB | SB |
| 1 | SC | SC |
| 1 | SD | SD |

表6.2の状態遷移表に現れる状態を、表6.1によって符号に置き換えることで、次状態関数回路 next_state_generator モジュールの真理値表が得られます(表6.3)。

<表6.3 次状態関数回路 next_state_generatorの真理値表>

| 入力 p | 入力 state[1:0] | 出力 next_state[1:0] |
|---|--------------|----------------|
| 0 | 00 | 01 |
| 0 | 01 | 10 |
| 0 | 10 | 11 |
| 0 | 11 | 00 |
| 1 | 00 | 00 |
| 1 | 01 | 01 |
| 1 | 10 | 10 |
| 1 | 11 | 11 |

表6.3の真理値表をもとにして、 next_state_generator モジュールはリスト6.2のように記述できます。

<リスト6.2 next_state_generator モジュール(次状態関数回路)>

```SystemVerilog : next_state_generator.sv
module next_state_generator (
  input   logic [1:0] state,
  input   logic       p,
  output  logic [1:0] next_state
);

  always_comb begin
    case ({p, state})
      3'b0_00:  next_state = 2'b01;
      3'b0_01:  next_state = 2'b10;
      3'b0_10:  next_state = 2'b11;
      3'b0_11:  next_state = 2'b00;
      3'b1_00:  next_state = 2'b00;
      3'b1_01:  next_state = 2'b01;
      3'b1_10:  next_state = 2'b10;
      3'b1_11:  next_state = 2'b11;
      default:  next_state = 2'b00;
    endcase
  end

endmodule //
```


### 出力関数回路

図6.2の状態遷移図をもとに、今の状態と出力の関係を表にまとめると、表6.4が得られます。

<表6.4 状態と出力の対応表>

| 今の状態 | 出力 y[1:0] |
|----------|-------------|
| SA | 00 |
| SB | 01 |
| SC | 00 |
| SD | 10 |

先ほどと同様に、状態を符号に置き換えることで、表6.5のように出力関数回路 output_decoder の真理値表が得られます。

<表6.5 出力関数回路 output_decoder の真理値表>

| 入力 state[1:0] | 出力 y[1:0] |
|-----------------|-------------|
| 00 | 00 |
| 01 | 01 |
| 10 | 00 |
| 11 | 10 |

表6.5より output_decoder モジュールはリスト6.3のように記述できます。

<リスト6.3 output_decoder モジュール(出力関数回路)>

```SystemVerilog : output_decoder.sv
module output_decoder (
  input   logic [1:0] state,
  output  logic [1:0] y
);

  always_comb begin
    case (state)
      2'b00   :  y = 2'b00;
      2'b01   :  y = 2'b01;
      2'b10   :  y = 2'b00;
      2'b11   :  y = 2'b10;
      default :  y = 2'b00;
    endcase
  end

endmodule //
```

### 状態機械の組み上げ

以上で、状態レジスタ register モジュール、次状態関数回路 next_state_generator モジュール、出力関数回路 output_decoder が設計できました。
最後にこれらの各モジュールを図6.3のように接続することで、状態機械 my_stm を作ることができます。

my_stm モジュールの設計例をリスト6.4に示します。

<リスト6.4 my_stm モジュール>

```SystemVerilog : my_stm.sv
module my_stm (
  input   logic       clock,
  input   logic       n_reset,
  input   logic       p,
  output  logic [1:0] y
);

  logic [1:0] state;      // 今の状態
  logic [1:0] next_state; // 次の状態

  // 状態レジスタ
  register state_register(
    .clock    (clock),
    .n_reset  (n_reset),
    .d        (next_state),
    .q        (state)
  );

  // 次状態関数
  next_state_generator next_state_generator(
    .p          (p),
    .state      (state),
    .next_state (next_state)
  );

  // 出力関数
  output_decoder output_decoder(
    .state      (state),
    .y          (y)
  );

endmodule //
```

### 演習

リスト6.4 my_stm モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。
my_stm モジュールの入出力信号は表6.6のように DE0-CV の入出力デバイスに割り当ててください。

なお、リスト6.1 register モジュール、リスト6.2 next_state_generator モジュール、およびリスト6.3 output_decoder モジュールも必要となりますので、プロジェクトにそれらのデザインファイルも追加しましょう。

<表6.6 my_stm モジュールの入出力のデバイスへの割り当て>

|信号名|割り当てデバイス|入出力|
|------|----------------|------|
|clock            | KEY0        | input |
|n_reset          | KEY1        | input |
|p                | SW0         | input |
|y[1:0]           | LEDR1-LEDR0 | output |


## 6.2 列挙型(enum)を用いたステートマシンの設計

ここでは、列挙型(enum)を用いてステートマシンの状態を定義し、より可読性の高いコードでステートマシンを設計する方法を学びます。

### 演習

## 6.3 Mealy 型ステートマシンの設計

Mealy 型ステートマシンは、出力が現在の状態と入力に依存するステートマシンです。
図 6.3 に示すような Mealy 型ステートマシン my_stm2 を設計します。

![ステートマシン my_stm2](./assets/my_stm2_circuit.svg "ステートマシン my_stm2")

<図6.3 my_stm2 の状態遷移図>


### 演習


## 6.4 課題

