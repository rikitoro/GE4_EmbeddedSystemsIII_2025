# 5章 レジスタの設計

本章では、always_ff 文を用いてレジスタを設計する方法を学びます。
always_ff 文は、クロック信号の立ち上がりや立ち下がりなど、特定のタイミングで動作する回路を記述できます。
この文を使用することで、フリップフロップの動作を簡潔に記述でき、レジスタやカウンタなどの同期回路を設計することが可能になります。


## 5.1 4ビットレジスタ

図 5.1a に示すような 4 ビットのレジスタを設計します。
このレジスタはクロック信号 `clock` の立ち上がりのタイミングで、
4ビットの入力信号 `d[3:0]` を取り込んで 4 ビットの出力信号 `q[3:0]` に出力し、
その値を次の `clock` の立ち上がりが入ってくるまで、そのまま保持する回路です。

![register4](./assets/chap05_register4.svg)

<図 5.1a 4ビットレジスタ>

このような 4 ビットレジスタは、リスト 5.1 のように always_ff を用いて記述できます。

<リスト 5.1 register4 モジュール>

```sv : register4.sv
module register4(
  input   logic         clock,
  input   logic   [3:0] d,
  output  logic   [3:0] q
);

  always_ff @ (posedge clock) begin  // clock の立ち上がりで動作
    q <= d;   // d の値を q に代入 (ノンブロッキング代入)
  end
endmodule
```

always_ff 文の部分を抜き出すと以下のようになります。

```sv
always_ff @ (posedge clock) begin // clock の立ち上がりで動作
  q <= d;   // d の値を q に代入 (ノンブロッキング代入)
end
```

ここでは、`posedge clock` というイベントが発生したときに、`q <= d;` の処理が実行されます。
`posedge clock` は、`clock` 信号の立ち上がりエッジを意味します。
すなわち、`clock` 信号が 0 から 1 に変化したときに、always_ff 文が起動し `q` に `d` の値が代入されます。

`posedge clock` の代わりに `negedge clock` を使用すると、`clock` 信号の立ち下がりエッジで動作することになります。

なお、`q <= d;` の部分はノンブロッキング代入と呼ばれ、`d` の値を `q` に代入することを意味します。
ノンブロッキング代入は、複数の代入が同時に実行される場合でも、各文が独立して動作することを保証します。
always_ff 文による同期式回路の設計ではノンブロッキング代入を使用することが推奨されます。


この 4 ビットレジスタの動作例をタイムチャートで示すと図 5.1b のようになります。

![register4_timing](./assets/chap05_register4_timing.svg)

<図 5.1b 4ビットレジスタ register4 モジュールの動作例>


この4ビットレジスタ register4 モジュールを呼び出して使うには次のように記述します。

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic [3:0] SW,     // d[3:0]
  output  logic [3:0] LEDR    // q[3:0]
);

  register4 reg4(
    .clock (KEY0),
    .d     (SW),
    .q     (LEDR)
  );
```

### 演習

4ビットレジスタを実習ボード DE0-CV に実装して、その動作を確かめてみましょう。
上記の shell モジュールをプロジェクトの Top level design entity として設定し、
その入出力を表 5.1 に示すように設定してください。

<表 5.1 shell モジュールの入出力のデバイスへの割り当て>

| 信号名    |割り当てデバイス| 入出力 |
|-----------|----------------|--------|
| KEY0      | KEY0           | input  |
| SW[3:0]   | SW3-SW0        | input  |
| LEDR[3:0] | LEDR3-LEDR0    | output |

ここで clock 信号を入力するために使用しているプッシュスイッチの KEY0 は、
ボタンを押し下げている間は 0 となり、離すと 1 となります。
KEY1 から KEY3 のプッシュスイッチについても同様です。
また、プッシュスイッチ KEY0 から KEY3 にはチャタリング防止のための回路が組み込まれています。
(なお、スライドスイッチ SW9 ~ SW0 にはチャタリング防止回路はついていません。)

レジスタの出力が変化するのは、KEY0 を押し下げた時なのか、それとも離した時なのか、そのタイミングを注意深く観察してください。


---

## 5.2 レジスタ (ビット幅のパラメータ化)

リスト 5.2 の register モジュールは、ビット幅をパラメータとして指定できるようにしたレジスタの設計例です。


<リスト 5.2 register モジュール>

```sv : register.sv
module register #(parameter WIDTH = 4)( // パラメータ WIDTH の定義 デフォルト値は 4
  input   logic             clock,
  input   logic [WIDTH-1:0] d,      // パラメータ WIDTH によるビット幅の指定 
  output  logic [WIDTH-1:0] q       // パラメータ WIDTH によるビット幅の指定
);

  always_ff @ (posedge clock) begin
    q <= d;
  end
endmodule
```

モジュールの定義において、モジュール名 `register` の後に `#(parameter WIDTH = 4)` と記述することで、`WIDTH` というパラメータを定義しています。
また `WIDTH` のデフォルト値を 4 としています。
このパラメータ `WIDTH` を使用して、入力 `d` と出力 `q` のビット幅を指定しています。　
このようにすることで、モジュールを呼び出す際にビット幅を変更することができます。

モジュールを呼び出す際には、次のようにパラメータの値を指定します。

```sv : shell.sv
module shell(
  input   logic       KEY0,
  input   logic [7:0] SW,
  output  logic [7:0] LEDR
);

  register #(.WIDTH(8)) reg8(　// パラメータ WIDTH を 8 に指定
    .clock  (KEY0),
    .d      (SW),
    .q      (LEDR)
  ); 
endmodule
```

ここでは `register` モジュールを呼び出す際に `#(.WIDTH(8))` と記述することで、パラメータ `WIDTH` の値を 8 に指定しています。
これにより、register モジュールの入力 `d` と出力 `q` のビット幅がそれぞれ 8 ビットになり、8 ビットのレジスタを作成することができます。

### 演習

上記の 8 ビットレジスタを搭載した shell モジュールを実習ボード DE0-CV に実装して、その動作を確かめてみましょう。
shell モジュールをプロジェクトの Top level design entity として設定し、その入出力を表 5.2 に示すように設定してください。

<表 5.2 shell モジュールの入出力のデバイスへの割り当て>

| 信号名    |割り当てデバイス| 入出力 |
|-----------|----------------|--------|
| KEY0      | KEY0           | input  |
| SW[7:0]   | SW7-SW0        | input  |
| LEDR[7:0] | LEDR7-LEDR0    | output |

---

## 5.3 同期リセット付きレジスタ

リスト 5.3 の register_r モジュールは、同期リセット機能を持つレジスタの設計例です。

<リスト 5.3 register_r モジュール>

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
```

この register_r モジュールは、2つのパラメータ `WIDTH` と `RESET_VALUE` を持ちます。
`WIDTH` はレジスタのビット幅を指定し、デフォルト値は 4 ビットです。
`RESET_VALUE` はリセット時に `q` に設定される値を指定し、デフォルト値は 0 です。
`RESET_VALUE` の設定値 `'0` (アポストロフィゼロ)は、左辺のビット幅に応じたゼロの値を意味します(ビット幅が自動的に設定されます)。

モジュール内の記述では always_ff 文を使用して、クロック `clock` の立ち上がりエッジで動作する回路を設計しています。
always 文の中では if 文を使用して条件により動作を変えることができます。
ここでは、`reset` が 1 のときは `q` にリセット値 `RESET_VALUE` を設定し、そうでない場合は `d` の値を `q` に設定しています。
このようにすることで、クロックの立ち上がりエッジで `reset` が 1 のときに `q` をリセット値に設定し、`reset` が 0 のときは `d` の値を保持することができます。
なお、この　register_r モジュールのリセットはclock の立ち上がりエッジで動作する同期リセットであることに注意しましょう。

register_r モジュールの動作例をタイムチャートで示すと図 5.3 のようになります。
ここでは、ビット幅 `WIDTH` を 4 ビット、リセット値 `RESET_VALUE` を 4'b0000 とした場合の例を示しています。
リセット信号 `reset` が 0 の時は、`clock` の立ち上がりエッジで入力 `d` の値が出力 `q` に設定されますが、
`reset` が 1 の時は、`clock` の立ち上がりエッジで出力 `q` にリセット値 `0000` が設定されます。
リセット動作も含め、出力 `q` の値が変化するのは `clock` の立ち上がりエッジのタイミングだけであることに注意してください。

![register_r_timing](./assets/chap05_register_r_timing.svg)

<図 5.3 register_r モジュールの動作例>

この register_r モジュールを呼び出して使うには次のように記述します。
モジュール呼び出し時にパラメータ値を設定しないと、デフォルト値が使用されます。
すなわち、ビット幅 `WIDTH` は 4 ビット、リセット値 `RESET_VALUE` は `4'b0000` となります。

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  input   logic [3:0] SW,     // d[3:0]
  output  logic [3:0] LEDR    // q[3:0]
);

  register_r reg_r( // ビット幅 4 ビット リセット値は 4'b0000 のレジスタ
    .clock    (KEY0),
    .reset    (SW9),
    .d        (SW),
    .q        (LEDR)
  );

endmodule
```

### 演習 1

上記の register_r モジュールを搭載した shell モジュールを実習ボード DE0-CV に実装して、その動作を確かめてみましょう。
shell モジュールをプロジェクトの Top level design entity として設定し、その入出力を表 5.3 に示すように設定してください。

<表 5.3 shell モジュールの入出力のデバイスへの割り当て>
| 信号名    |割り当てデバイス| 入出力 |
|-----------|----------------|--------|
| KEY0      | KEY0           | input  |
| SW9       | SW9            | input  |
| SW[3:0]   | SW3-SW0        | input  |
| LEDR[3:0] | LEDR3-LEDR0    | output |

### 演習 2

register_r モジュールのリセット値 `RESET_VALUE` を `4'b1111` に変更して、レジスタの出力がリセット時に `4'b1111` になるようにしてみましょう。
次の shell モジュールのように、register_r モジュールの呼び出しの際のパラメータ値を変更してみてください。

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  input   logic [3:0] SW,
  output  logic [3:0] LEDR
);

  register_r #(.RESET_VALUE (4'b1111)) reg_r( // リセット値 RESET_VALUE を 4'b1111 に設定
    .clock    (KEY0),
    .reset    (SW9),
    .d        (SW),
    .q        (LEDR)
  );

endmodule
```

この shell モジュールを実習ボード DE0-CV に実装して、その動作を確かめてみましょう。
先ほどの演習と同様に、プロジェクトの Top level design entity として設定し、
その入出力を表 5.3 に示すように設定してください。


--- 

## 5.4 書き込み許可付きレジスタ


```sv : register_r_en.sv
module register_r_en #(
  parameter WIDTH = 4,
  parameter logic [WIDTH-1:0] RESET_VALUE = '0
)(
  input   logic             clock,
  input   logic             reset,
  input   logic             en,      // write-enable
  input   logic [WIDTH-1:0] d,
  output  logic [WIDTH-1:0] q
);

  always_ff @ (posedge clock) begin
    if (reset == 1'b1) begin
      q <= RESET_VALUE;
    end else if (en == 1'b1) begin 
      q <= d;
    end
      
  end
  
endmodule
```


```sv : shell.sv 
module shell(
  input   logic       KEY0,   // clock
  input   logic       KEY3,   // reset (active-low)
  input   logic       SW9,    // write-enable
  input   logic [7:0] SW,
  output  logic [7:0] LEDR
);

  register_r_en reg_r(
    .clock  (KEY0),
    .reset  (KEY3),
    .en     (SW9),
    .d      (SW),
    .q      (LEDR)
  );
 
endmodule
```

### 演習 

---

## 5.5 10進カウンタ

```sv : counter10.sv
module counter10(
  input   logic       clock,
  input   logic       reset,
  output  logic [3:0] count
);

  logic [3:0] count_next;

  assign count_next = (count == 4'd9) ? 4'd0 : count + 1'd1;

  regiser_r reg(
    .clock  (clock),
    .reset  (reset),
    .d      (count_next),
    .q      (count)
  );

endmodule
```

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  output  logic [3:0] LEDR
);

  counter10 counter(
    .clock  (KEY0),
    .reset  (SW9),
    .count  (LEDR)
  );
endmodule
```

### 演習

---

## 5.6 N 進カウンタ

```sv : counterN.sv
module counterN #(
  parameter WIDTH = 4,
  parameter [WIDTH-1:0] N = '1
)(
  input   logic             clock,
  input   logic             reset,
  output  logic [WIDTH-1:0] count
);

  logic [WIDTH-1:0] count_next;

  assign count_next = (count == N-1) ? '0 : count + 1'd1;

  register_r #(.WIDTH (WIDTH)) reg(
    .clock  (clock),
    .reset  (reset),
    .d      (count_next),
    .q      (count)
  );

endmodule
```

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  output  logic [6:0] HEX1,
  output  logic [6:0] HEX0
);
  
  logic [5:0] count;

  counterN #(.WIDTH(8), .N(42)) counter42(
    .clock  (KEY0),
    .reset  (SW9),
    .count  (count)
  );

  // 7セグメントディスプレイへの変換
  sseg_decoder dec1(
    .num  ({2'b00, count[5:4]})
    .hex  (HEX1)
  );

  sseg_decoder dec0(
    .num  (count[3:0]),
    .hex  (HEX0)
  );

endmodule
```

### 演習

---