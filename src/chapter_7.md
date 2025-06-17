# 6章 ステートマシンの設計

前章までで、レジスタの設計方法と組み合わせ回路の設計方法、さらにそれらの回路モジュールを組み合わせて新しい回路を設計する方法を学んできました。
レジスタと組み合わせ回路とを組み合わせることで、ステートマシン(有限状態機械)を設計することができます。
ステートマシンは内部に状態を持ち、入力に応じて状態を遷移し、出力を生成する回路です。
ステートマシンは、コンピュータのCPUや、通信プロトコル、制御回路など、様々な分野で広く利用されています。
ステートマシンの設計方法を学ぶことで、より複雑な回路を設計できるようになります。


## 6.1 Moore 型ステートマシンの設計

Moore 型ステートマシンは、出力が現在の状態のみに依存するステートマシンです。
状態遷移は入力と現在の状態に基づいて次の状態が決まりますが、出力は現在の状態にのみ依存します。

ここではステートマシンの例として、図 6.1a に示すような 1 bit の入力 p、クロック入力 clock、同期リセット信号 reset、1 bit の出力 y を持つステートマシン my_stm1を設計します。

![ステートマシン my_stm1](./assets/my_stm1.svg "ステートマシン my_stm")

<図6.1a ステートマシン my_stm1 とその状態遷移図>

ステートマシン my_stm1 の状態遷移は図 6.1a の右側に示す状態遷移図で与えます。
my_stm1 は 3 つの状態(SA, SB, SC)を持ち、出力 y が状態だけで決まるような Moore 型のステートマシンです。
状態遷移は clock の立ち上がりのタイミングで起こるものとします。
同期リセット信号 reset が 1 になると、clock の立ち上がりで状態は SA にリセットされます。

my_stm1 の期待される動作例を図 6.1b にタイムチャートで示します。
(状態は外部からは見えないので、破線で示しています。)

![my_stm1 の動作例](./assets/timechart_my_stm1.svg "my_stm1 の動作例")

<図6.1b my_stm1 の動作例>

クロックの立ち上がりで状態が遷移し、その状態に応じて出力 y が変化することがわかります。

以下ではこのステートマシン my_stm1 を設計していきます。

### 状態の符号化

ステートマシン my_stm1 は 3 つの状態を持ちます。
ステートマシンを実現するには、現在どの状態にあるかを表現し保持するために、取りうる状態を 2 進数のビット列で符号化する必要があります。
状態の符号化にはいろいろな方法がありますが、ここでは one-hot 符号化と呼ばれる方法を用いることにします。

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

<リスト6.1a register_r モジュール(同期リセット付きレジスタ)>

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

ステートマシン my_stm1 の状態は 3 ビットで表現され、リセット時には状態 SA にリセットされます。
したがって次のように register_r モジュールについて、そのパラメータを `WIDTH = 3`, `RESET_VALUE = 3'b100` と設定してインスタンス化することで、状態レジスタを作成できます。

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

### 次状態生成回路の設計

次状態生成回路は、現在の状態と入力に基づいて次の状態を決定する組み合わせ回路です。

状態生成回路を設計するために、まず、図 6.1a の状態遷移図をもとに、現在の状態と入力 p の組み合わせから次の状態を決定する状態遷移表を作成します。

<表6.1b my_stm1 の状態遷移表>

| 現在の状態 | 入力 p | 次の状態 |
|----------|--------|----------|
| SA       | 0      | SA       |
| SA       | 1      | SB       |
| SB       | 0      | SC       |
| SB       | 1      | SA       |
| SC       | 0      | SC       |
| SC       | 1      | SA       |


現在の状態と次の状態を表 6.1a によって符号化することで次状態生成回路の真理値表を得ることができます。
以降では、次状態生成回路の入力となる現在の状態を state, 出力となる次の状態を next_state で表すこととします。
表 6.1c に、符号化後の次状態生成回路の真理値表を示します。
なお、状態遷移表に現れない入力値の組み合わせ(例えば state = 011, p = 0)に対しては、デフォルトで SA にリセットされるようにしています。



<表6.1c my_stm1 の次状態生成回路の真理値表>

| state | p | next_state |
|-------|---|------------|
| 100   | 0 | 100        |
| 100   | 1 | 010        |
| 010   | 0 | 001        |
| 010   | 1 | 100        |
| 001   | 0 | 001        |
| 001   | 1 | 100        |
| その他 | * | 100       |


リスト 6.1b に、次状態生成回路 next_state_generator モジュールの HDL 記述を示します。
always_comb 文の中で case 文を用いて、表 6.1c の入出力の関係を組合せ回路として記述しています。
なお、表に現れない入力値の組み合わせ(例えば state = 011, p = 0)に対しては、デフォルトで SA にリセットされるようにしています。

<リスト 6.1b next_state_generator モジュール(次状態関数回路)>

```sv : next_state_generator.sv
module next_state_generator (
  input   logic [2:0] state,      // 現在の状態
  input   logic       p,          // 入力
  output  logic [2:0] next_state  // 次の状態
);

  always_comb begin
    case ({state, p}) // state と p を連結して状態遷移を定義
      4'b100_0: next_state = 3'b100; // SA -- p=0 --> SA
      4'b100_1: next_state = 3'b010; // SA -- p=1 --> SB
      4'b010_0: next_state = 3'b001; // SB -- p=0 --> SC
      4'b010_1: next_state = 3'b100; // SB -- p=1 --> SA
      4'b001_0: next_state = 3'b001; // SC -- p=0 --> SC
      4'b001_1: next_state = 3'b100; // SC -- p=1 --> SA
      default:  next_state = 3'b100; // デフォルトは SA にリセット
    endcase
  end

endmodule
```

### 出力生成回路の設計

出力生成回路は、現在の状態に基づいて出力を生成する組み合わせ回路です。

図6.1aの状態遷移図から、各状態における出力 y の値を表 6.1d に示します。

<表6.1d 状態と出力の対応表>

| 今の状態 | 出力 y |
|----------|--------|
| SA       | 0      |
| SB       | 1      |
| SC       | 0      |

先ほどの次状態生成回路と同様に、状態を符号に置き換えることで、表 6.1e に示すような出力生成回路 output_decoder の真理値表が得られます。
状態と出力の対応表に現れない入力値の組み合わせ(例えば state = 011)に対しては、デフォルトで出力 0 としています。

<表6.1e 出力生成回路の真理値表>

| state | y |
|-------|---|
| 100   | 0 |
| 010   | 1 |
| 001   | 0 |
| その他 | 0 |


リスト 6.1c に、出力生成回路 output_decoder モジュールの HDL 記述を示します。
always_comb 文の中で case 文を用いて、表 6.1e の入出力の関係を組合せ回路として記述しています。
なお、表に現れない入力値の組み合わせ(例えば state = 011)に対しては、デフォルトで出力 0 としています。

<リスト 6.1c output_decoder モジュール(出力生成回路)>

```sv : output_decoder.sv
module output_decoder (
  input   logic [2:0] state, // 今の状態
  output  logic       y      // 出力
);

  always_comb begin
    case (state)
      3'b100: y = 1'b0; // SA のときは出力 0
      3'b010: y = 1'b1; // SB のときは出力 1
      3'b001: y = 1'b0; // SC のときは出力 0
      default: y = 1'b0; // デフォルトは出力 0
    endcase
  end

endmodule
```

### ステートマシンの組み上げ

以上で、状態レジスタ register_r モジュール、次状態生成回路 next_state_generator モジュール、出力生成回路 output_decoder を設計することができました。

これらの各モジュールを図 6.1c のように接続することで、ステートマシン my_stm1 を構築することができます。

![ステートマシン my_stm1 の構成図](./assets/my_stm1_circuit.svg "ステートマシン my_stm1 の構成図")

<図6.1c my_stm1 の構成図>


この構成図をもとに作成した my_stm1 モジュールの HDL 記述をリスト 6.1d に示します。


<リスト 6.1d my_stm モジュール>

```sv : my_stm1.sv
module my_stm1 (
  input   logic       clock,
  input   logic       reset,
  input   logic       p,
  output  logic       y
);

  logic [2:0] state;      // 現在の状態
  logic [2:0] next_state; // 次の状態

  // 状態レジスタ
  register_r #(
    .WIDTH(3),            // 3 ビットのレジスタ
    .RESET_VALUE(3'b100)  // SA にリセット
  ) state_register (
    .clock(clock),
    .reset(reset),
    .d(next_state),   // 次の状態を入力
    .q(state)         // 現在の状態を出力
  );

  // 次状態関数
  next_state_generator next_state_gen(
    .p          (p),
    .state      (state),        // 現在の状態を入力
    .next_state (next_state)    // 次の状態を出力
  );

  // 出力関数
  output_decoder output_dec(
    .state      (state),    // 現在の状態を入力
    .y          (y)         // 出力を出力 
  );

endmodule //
```

### 演習

my_stm1 モジュールを搭載した以下の shell モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。
以下の shell モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。
```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,   // reset
  input   logic       SW0,    // p
  output  logic       LEDR0   // y
);

  my_stm1 stm(
    .clock(KEY0),   // clock
    .reset(SW9),    // reset
    .p(SW0),        // p
    .y(LEDR0)       // y
  );

endmodule
```

shell モジュールを top level design entity としてプロジェクトに設定し、入出力信号を表 6.1f のように割り当ててください。

<表 6.1f my_stm1 モジュールの入出力のデバイスへの割り当て>

|信号名|割り当てデバイス|入出力 |
|------|----------------|-------|
|clock | KEY0           | input |
|reset | SW9            | input |
|p     | SW0            | input |
|y     | LEDR0          | output|

プロジェクトには shell モジュールに加えて、以下のモジュールを定義したファイルが必要となることに注意してください。
- my_stm1 (リスト 6.1d のステートマシン)
- register_r  (リスト 6.1a の同期リセット付きレジスタ)
- next_state_generator (リスト 6.1b の次状態生成回路)
- output_decoder (リスト 6.1c の出力生成回路)



## 6.2 列挙型(enum)を用いたステートマシンの設計

ここでは、列挙型(enum)を用いてステートマシンの状態を定義し、より可読性の高いコードでステートマシンを設計する方法を学びます。
列挙型(enum)を用いることで、状態を数値ではなく意味のある名前で表現でき、コードの可読性が向上します。
列挙型を用いたステートマシンの設計方法を学ぶために、前節で設計した my_stm1 モジュールを列挙型を用いて書き直してみます。

列挙型は SystemVerilog で定義できるデータ型の一つで、特定の値の集合を名前付きで定義することができます。

列挙型の定義は次のように行います。

```sv :
  typedef enum logic [1:0] {
    SA = 2'b00, // 状態 SA
    SB = 2'b01, // 状態 SB
    SC = 2'b10  // 状態 SC
  } state_t;
```

ここでは、列挙型 `state_t` を定義し、状態 SA, SB, SC をそれぞれ 2 ビットの値で表現しています。
なお、7.1 節では、状態を 3 ビットで符号化していましたが、ここでは 2 ビット幅の列挙型を用いて符号化しています。
(もちろん、3 ビットで符号化しても問題ありません。)

以下のリスト 6.2 に、列挙型を用いて my_stm1 モジュールを再設計した my_stm2 モジュールの HDL 記述を示します。

<リスト 6.2 列挙型を用いた my_stm2 モジュール>

```sv : my_stm2.sv
module my_stm2 (
  input   logic       clock,
  input   logic       reset,
  input   logic       p,
  output  logic       y
);

  // 列挙型で状態を定義
  typedef enum logic [1:0] {
    SA = 2'b00, // 状態 SA
    SB = 2'b01, // 状態 SB
    SC = 2'b10  // 状態 SC
  } state_t;

  state_t state;      // 現在の状態
  state_t next_state; // 次の状態

  // 状態レジスタ
  register_r #(
    .WIDTH(2),           // 列挙型のビット幅は 2
    .RESET_VALUE(SA)     // SA にリセット
  ) state_register (
    .clock(clock),
    .reset(reset),
    .d(next_state),   // 次の状態を入力
    .q(state)         // 現在の状態を出力
  );

  // 次状態生成回路部分
  always_comb begin
    case ({state, p}) // state と p を連結して状態遷移を定義
      {SA, 1'b0}: next_state = SA; // SA -- p=0 --> SA
      {SA, 1'b1}: next_state = SB; // SA -- p=1 --> SB
      {SB, 1'b0}: next_state = SC; // SB -- p=0 --> SC
      {SB, 1'b1}: next_state = SA; // SB -- p=1 --> SA
      {SC, 1'b0}: next_state = SC; // SC -- p=0 --> SC
      {SC, 1'b1}: next_state = SA; // SC -- p=1 --> SA
      default:    next_state = SA; // デフォルトは SA にリセット
    endcase
  end

  // 出力生成回路部分
  always_comb begin
    case (state)
      SA: y = 1'b0; // SA のときは出力 0
      SB: y = 1'b1; // SB のときは出力 1
      SC: y = 1'b0; // SC のときは出力 0
      default: y = 1'b0; // デフォルトは出力 0
    endcase
  end
endmodule
```

この my_stm2 モジュールは、列挙型を用いて状態を定義し、状態遷移と出力生成を行っています。
現在の状態と次の状態を表す内部信号 `state` と `next_state` は、列挙型 `state_t` を用いて定義されています。
また、case 文を用いて、状態遷移と出力信号の生成を行っていますが、このパターンマッチの部分にも列挙型 `state_t` の値 (SA, SB, SC) を用いることで、より可読性の高いコードになっています。


### 演習

my_stm2 モジュールを搭載した以下の shell モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  input   logic       SW0,    // p
  output  logic       LEDR0   // y
);

  my_stm2 stm(
    .clock(KEY0),   // clock
    .reset(SW9),    // reset
    .p(SW0),        // p
    .y(LEDR0)       // y
  );
endmodule
``` 

shell モジュールを top level design entity としてプロジェクトに設定し、入出力信号を表 6.2 のように割り当ててください。

<表 6.2 my_stm2 モジュールの入出力のデバイスへの割り当て>

|信号名|割り当てデバイス|入出力 |
|------|----------------|-------|
|clock | KEY0           | input |
|reset | SW9            | input | 
|p     | SW0            | input |
|y     | LEDR0          | output|

プロジェクトには shell モジュールに加えて、以下のモジュールを定義したファイルが必要となることに注意してください。
- my_stm2 (リスト 6.2 のステートマシン) 
- register_r  (リスト 6.1a の同期リセット付きレジスタ)




## 6.3 Mealy 型ステートマシンの設計

Mealy 型ステートマシンは、出力が現在の状態と入力に依存するステートマシンです。
ここでは、図 6.3a に示すような Mealy 型ステートマシン my_stm3 を設計します。

![ステートマシン my_stm3](./assets/my_stm3.svg "ステートマシン my_stm3")

<図 6.3a Mealy 型ステートマシン my_stm3>

my_stm3 は 2 つの状態(SA, SB)を持ち、入力 p と現在の状態に応じて出力 y が決まる Mealy 型のステートマシンです。
状態遷移は clock の立ち上がりのタイミングで起こるものとします。
同期リセット信号 reset が 1 になると、clock の立ち上がりで状態は SA にリセットされます。

my_stm3 の期待される動作例を図 6.3b にタイムチャートで示します。
(状態は外部からは見えないので、破線で示しています。)
出力 y は、状態が SB で、かつ、入力 p が 1 のときに 1 となることに注意してください。

![my_stm3 の動作例](./assets/timechart_my_stm3.svg "my_stm3 の動作例")


状態遷移図をもとに、my_stm3 の状態遷移表を作成すると次のようになります。
Moore 型ステートマシンと同様に、次の状態は現在の状態と入力 p によって決まります。

<表 6.3a my_stm3 の状態遷移表>

| 現在の状態 | 入力 p | 次の状態 |
|----------|--------|----------|
| SA       | 0      | SA       |
| SA       | 1      | SB       |
| SB       | 0      | SB       |
| SB       | 1      | SB       |


Mealy 型ステートマシンでは、出力 y は現在の状態と入力 p によって決まります。
対応表を作成すると次のようになります。

<表 6.3b my_stm3 の入力、状態と出力との対応表>

| 現在の状態 | 入力 p | 出力 y |
|----------|--------|--------|
| SA       | 0      | 0      |
| SA       | 1      | 0      |
| SB       | 0      | 0      |
| SB       | 1      | 1      |

my_stm3 は 2 つの状態(SA, SB)を持ちますが、これらは列挙型を用いて符号化することができます。
列挙型を用いて状態を定義すると次のようになります。
(状態が2つしかないので、1 ビットの列挙型で定義することができます。)

```sv :
  typedef enum logic {
    SA = 1'b0, // 状態 SA
    SB = 1'b1  // 状態 SB
  } state_t;
```

この列挙型を用いて、my_stm3 モジュールを設計します。
表 6.3a の状態遷移表と表 6.3b の出力の対応表をもとに作成した、my_stm3 モジュールの HDL 記述をリスト 6.3 に示します。

<図 6.3b my_stm3 の動作例>

```sv : my_stm3.sv
module my_stm3 (
  input   logic       clock,
  input   logic       reset,
  input   logic       p,
  output  logic       y
);

  // 列挙型で状態を定義
  typedef enum logic {
    SA = 1'b0, // 状態 SA
    SB = 1'b1  // 状態 SB
  } state_t;

  state_t state;      // 現在の状態
  state_t next_state; // 次の状態

  // 状態レジスタ
  register_r #(
    .WIDTH(1),           // 列挙型のビット幅は 1
    .RESET_VALUE(SA)     // SA にリセット
  ) state_register (
    .clock(clock),
    .reset(reset),
    .d(next_state),   // 次の状態を入力
    .q(state)         // 現在の状態を出力
  );

  // 次状態生成回路
  always_comb begin
    case ({state, p}) // state と p を連結して状態遷移を定義
      {SA, 1'b0}: next_state = SA; // SA -- p=0 --> SA
      {SA, 1'b1}: next_state = SB; // SA -- p=1 --> SB
      {SB, 1'b0}: next_state = SB; // SB -- p=0 --> SB
      {SB, 1'b1}: next_state = SA; // SB -- p=1 --> SA
      default:    next_state = SA; // デフォルトは SA にリセット
    endcase
  end

  // 出力生成回路 (Mealy 型)
  always_comb begin
    case ({state, p}) // state と p を連結して出力を定義
      {SA, 1'b0}: y = 1'b0; // SA, p=0 のときは y=0
      {SA, 1'b1}: y = 1'b0; // SA, p=1 のときは y=0
      {SB, 1'b0}: y = 1'b0; // SB, p=0 のときは y=0
      {SB, 1'b1}: y = 1'b1; // SB, p=1 のときは y=1
      default:    y = 1'b0; // デフォルトは y = 0
    endcase
  end
endmodule
```

### 演習

my_stm3 モジュールを搭載した以下の shell モジュールを実習ボード DE0-CV に実装してその動作を確認しましょう。

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  input   logic       SW0,    // p
  output  logic       LEDR0   // y
);

  my_stm3 stm(
    .clock(KEY0),   // clock
    .reset(SW9),    // reset
    .p(SW0),        // p
    .y(LEDR0)       // y
  );
endmodule
```

shell モジュールを top level design entity としてプロジェクトに設定し、入出力信号を表 6.3c のように割り当ててください。
<表 6.3c my_stm3 モジュールの入出力のデバイスへの割り当て>

|信号名|割り当てデバイス|入出力 |
|------|----------------|-------|
|clock | KEY0           | input |
|reset | SW9            | input |
|p     | SW0            | input |
|y     | LEDR0          | output|

プロジェクトには shell モジュールに加えて、以下のモジュールを定義したファイルが必要となることに注意してください。
- my_stm3 (リスト 6.3 のステートマシン)
- register_r  (リスト 6.1a の同期リセット付きレジスタ)


## 6.4 課題

図 6.4 に示すような 2ビットの入力 p と 7 ビットの出力 HEX を持つ Moore 型のステートマシン fib_stm を設計しましょう。
状態遷移は clock の立ち上がりのタイミングで起こるものとします。
また、出力 HEX は 7 セグメントディスプレイに接続することを想定しています。
状態に応じて 7 セグメントディスプレイに 1, 2, 3 のパターンを表示させてください。

![ステートマシン fib_stm](./assets/fib_stm.svg "ステートマシン fib_stm")

<図 6.4 ステートマシン fib_stm>