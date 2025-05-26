# 5章 レジスタの設計

本章では、always_ff 文を用いてレジスタを設計する方法を学びます。
always_ff 文は、クロック信号の立ち上がりや立ち下がりなど、特定のタイミングで動作する回路を記述できます。
この文を使用することで、フリップフロップの動作を簡潔に記述でき、レジスタやカウンタなどの同期回路を設計することが可能になります。


## 5.1 4ビットレジスタ

```sv : register4.sv
module register4(
  input   logic         clock,
  input   logic   [3:0] d,
  output  logic   [3:0] q
);

  always_ff @ (posedge clock) begin  // clock の立ち上がりで動作
    q <= d;
  end
endmodule
```

### 演習

4ビットレジスタを実習ボード DE0-CV に実装して、その動作を確かめてみましょう。
register4 モジュールを top level design entity とし、
その入出力を表 5.1 に示すように設定してください。

<表 5.1 shell モジュールの入出力のデバイスへの割り当て>

| 信号名   |割り当てデバイス| 入出力 |
|----------|----------------|--------|
| clock    | KEY0           | input  |
| d[3:0]   | SW3-SW0        | input  |
| q[3:0]   | LEDR3-LEDR0    | output |

なお、clock 信号として使用しているプッシュスイッチの KEY0 は、
ボタンを押し下げている間は 0 となり、離すと 1 となります。
また、KEY0 から KEY3 にはチャタリング防止のための回路が組み込まれています。
(チャタリングとは、スイッチを切り替えたとき接点が何度も開閉する現象のことです。)

レジスタの出力が変化するのは、KEY0 を押し下げた時なのか、それとも離した時なのか、そのタイミングを注意深く観察してください。


---

## 5.2 レジスタ (ビット幅のパラメータ化)

リスト 5.1 の register モジュールは、ビット幅をパラメータとして指定できるようにしたレジスタの設計例です。

```sv : register.sv
module register #(parameter WIDTH = 4)(
  input   logic         clock,
  input   logic [WIDTH-1:0] d,
  output  logic [WIDTH-1:0] q
);

  always_ff @ (posedge clock) begin
    q <= d;
  end
endmodule
```

```sv : shell.sv
module shell(
  input   logic       KEY0,
  input   logic [7:0] SW,
  output  logic [7:0] LEDR
);

  register #(.WIDTH(8)) reg8(
    .clock  (KEY0),
    .d      (SW),
    .q      (LEDR)
  ); 
  
endmodule

```

---

## 5.3 同期リセット付きレジスタ

```sv : register_r.sv
module register_r #(
  parameter WIDTH = 4,
  parameter logic [WIDTH-1:0] RESET_VALUE = '0
)(
  input   logic             clock,
  input   logic             reset, // reset
  input   logic [WIDTH-1:0] d,
  output  logic [WIDTH-1:0] q
);

  always_ff @ (posedge clock) begin
    if (reset == 1'b1) begin
      q <= RESET_VALUE;
    end else begin 
      q <= d;
    end 
  end
```

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  input   logic [7:0] SW,
  output  logic [7:0] LEDR
);

  register_r reg_r(
    .clock    (KEY0),
    .reset    (SW9),
    .d        (SW),
    .q        (LEDR)
  );
endmodule
```

### 演習 1

### 演習 2

```sv : shell.sv
module shell(
  input   logic       KEY0,   // clock
  input   logic       SW9,    // reset
  input   logic [7:0] SW,
  output  logic [7:0] LEDR
);

  register_r #(.RESET_VALUE (4'b1111)) reg_r(
    .clock    (KEY0),
    .reset    (SW9),
    .d        (SW),
    .q        (LEDR)
  );

endmodule
```


--- 

## 5.4 書き込み許可付きレジスタ


```sv : register_r.sv
module register_r_en #(
  parameter WIDTH = 4,
  parameter logic [WIDTH-1:0] RESET_VALUE = '0
)(
  input   logic             clock,
  input   logic             reset, // reset (active-low)
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