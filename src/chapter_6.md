# 6章 レジスタを用いた応用回路の設計

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
  
  always_ff @ (negedge clock) begin
    if (reset == 1'b1) begin
      q <= RESET_VALUE;
    end else if (en == 1'b1) begin
      q <= d;
    end 
  end

endmodule
```

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