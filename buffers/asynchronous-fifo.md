---
description: >-
  What is FIFO and what is asynchronous FIFO? How does it work and why is it
  used?
---

# Asynchronous FIFO

{% embed url="https://zipcpu.com/blog/2018/07/06/afifo.html" %}
A really nice blog that explains & verifies the operation of a asynchronous FIFO
{% endembed %}

{% file src="../.gitbook/assets/CummingsSNUG2002SJ_FIFO1.pdf" %}

{% embed url="https://github.com/ZipCPU/website/blob/master/examples/afifo.v" %}
Async FIFO implementation reference
{% endembed %}

## Verilog implementation code (for reference)

```verilog
`default_nettype    none
//
//
module afifo(
  i_wclk, i_wrst_n, i_wr, i_wdata, o_wfull,
  i_rclk, i_rrst_n, i_rd, o_rdata, o_rempty
  );
    parameter   DSIZE = 8,  // 1 byte (8 bits)
                ASIZE = 5;  // 16 entries (2^5 = 32 entries)
    localparam	DW = DSIZE,
                AW = ASIZE;
    input	wire                i_wclk, i_wrst_n, i_wr;
    input	wire	[DW-1:0]    i_wdata;
    output	reg                 o_wfull;
    input	wire                i_rclk, i_rrst_n, i_rd;
    output	wire    [DW-1:0]    o_rdata;
    output	reg                 o_rempty;

    wire	[AW-1:0]    waddr, raddr;
    wire                wfull_next, rempty_next;
    reg	    [AW:0]      wgray, wbin, wq2_rgray, wq1_rgray,
                        rgray, rbin, rq2_wgray, rq1_wgray;
    
    wire    [AW:0]      wgraynext, wbinnext;
    wire    [AW:0]      rgraynext, rbinnext;

    reg     [DW-1:0]    mem	[0:((1<<AW)-1)];

    /////////////////////////////////////////////
    //
    //
    // Write logic
    //
    //
    /////////////////////////////////////////////

    //
    // Cross clock domains
    //
    // Cross the read Gray pointer into the write clock domain
    initial	{ wq2_rgray,  wq1_rgray } = 0;
    always @(posedge i_wclk or negedge i_wrst_n)
    if (!i_wrst_n)
        { wq2_rgray, wq1_rgray } <= 0;
    else
        { wq2_rgray, wq1_rgray } <= { wq1_rgray, rgray };
    
    
    // Calculate the next write address, and the next graycode pointer.
    assign	wbinnext  = wbin + { {(AW){1'b0}}, ((i_wr) && (!o_wfull)) };
    assign	wgraynext = (wbinnext >> 1) ^ wbinnext;

    assign	waddr = wbin[AW-1:0];

    // Register these two values--the address and its Gray code
    // representation
    initial	{ wbin, wgray } = 0;
    always @(posedge i_wclk or negedge i_wrst_n)
    if (!i_wrst_n)
        { wbin, wgray } <= 0;
    else
        { wbin, wgray } <= { wbinnext, wgraynext };

    assign	wfull_next = (wgraynext == { ~wq2_rgray[AW:AW-1],
                wq2_rgray[AW-2:0] });

    //
    // Calculate whether or not the register will be full on the next
    // clock.
    initial	o_wfull = 0;
    always @(posedge i_wclk or negedge i_wrst_n)
    if (!i_wrst_n)
        o_wfull <= 1'b0;
    else
        o_wfull <= wfull_next;

    //
    // Write to the FIFO on a clock
    always @(posedge i_wclk)
    if ((i_wr)&&(!o_wfull))
        mem[waddr] <= i_wdata;

    ////////////////////////////////////////////////////////////////////////
    ////////////////////////////////////////////////////////////////////////
    //
    //
    // Read logic
    //
    //
    ////////////////////////////////////////////////////////////////////////
    ////////////////////////////////////////////////////////////////////////
    //
    //

    //
    // Cross clock domains
    //
    // Cross the write Gray pointer into the read clock domain
    initial	{ rq2_wgray,  rq1_wgray } = 0;
    always @(posedge i_rclk or negedge i_rrst_n)
    if (!i_rrst_n)
        { rq2_wgray, rq1_wgray } <= 0;
    else
        { rq2_wgray, rq1_wgray } <= { rq1_wgray, wgray };


    // Calculate the next read address,
    assign	rbinnext  = rbin + { {(AW){1'b0}}, ((i_rd)&&(!o_rempty)) };
    // and the next Gray code version associated with it
    assign	rgraynext = (rbinnext >> 1) ^ rbinnext;

    // Register these two values, the read address and the Gray code version
    // of it, on the next read clock
    //
    initial	{ rbin, rgray } = 0;
    always @(posedge i_rclk or negedge i_rrst_n)
    if (!i_rrst_n)
        { rbin, rgray } <= 0;
    else
        { rbin, rgray } <= { rbinnext, rgraynext };

    // Memory read address Gray code and pointer calculation
    assign	raddr = rbin[AW-1:0];

    // Determine if we'll be empty on the next clock
    assign	rempty_next = (rgraynext == rq2_wgray);

    initial o_rempty = 1;
    always @(posedge i_rclk or negedge i_rrst_n)
    if (!i_rrst_n)
        o_rempty <= 1'b1;
    else
        o_rempty <= rempty_next;

    //
    // Read from the memory--a clockless read here, clocked by the next
    // read FLOP in the next processing stage (somewhere else)
    //
    assign	o_rdata = mem[raddr];
endmodule
```

* `DSIZE` is the width of the data ports in bits
  * A `DSIZE` f 8, means that it can store a byte (8 bits) of data
* `ASIZE` is the depth of the FIFO, how many entires it can hold
  * A `ASIZE` of 4, means there are `2^4=16` entires it can store
  * In other words, you can also think of it as the **address space**
  * Note: Don't set the address space too large, otherwise, compiling might take a long time (not sure if there would be enough resources too)

## References

* [https://vlsiverify.com/verilog/verilog-codes/asynchronous-fifo/](https://vlsiverify.com/verilog/verilog-codes/asynchronous-fifo/)
* [https://github.com/ujjwal-2001/Async\_FIFO\_Design](https://github.com/ujjwal-2001/Async_FIFO_Design)
* [https://zipcpu.com/blog/2018/07/06/afifo.html](https://zipcpu.com/blog/2018/07/06/afifo.html)
* [https://blog.csdn.net/Loudrs/article/details/131021317](https://blog.csdn.net/Loudrs/article/details/131021317)
