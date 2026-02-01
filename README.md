# week-3
```verilog
`timescale 1ns/1ps

module newfeature (
    input  wire       clk,
    input  wire       reset,
    input  wire       enable,

    input  wire [7:0] pixel_in,
    input  wire [7:0] pixel_left,
    input  wire [7:0] pixel_up,
    input  wire [7:0] pixel_up_left,
    input  wire [7:0] pixel_up_right,

    output reg  [7:0] confidence,
    output reg        fake_detected
);

    // --- internal scores ---
    reg [15:0] block_score;
    reg [15:0] hf_score;

    // --- running avg for auto threshold ---
    reg [23:0] sum_block;
    reg [23:0] sum_hf;
    reg [10:0] count_pix;

    // --- pixel differences ---
    wire [7:0] diff_h1 = (pixel_in > pixel_left)
                       ? (pixel_in - pixel_left)
                       : (pixel_left - pixel_in);

    wire [7:0] diff_v1 = (pixel_in > pixel_up)
                       ? (pixel_in - pixel_up)
                       : (pixel_up - pixel_in);

    wire [7:0] diff_d1 = (pixel_in > pixel_up_left)
                       ? (pixel_in - pixel_up_left)
                       : (pixel_up_left - pixel_in);

    wire [7:0] diff_d2 = (pixel_in > pixel_up_right)
                       ? (pixel_in - pixel_up_right)
                       : (pixel_up_right - pixel_in);

    // --- feature strengths ---
    wire [9:0] edge_strength = diff_h1 + diff_v1;
    wire [9:0] hf_strength   = diff_d1 + diff_d2;

    // --- adaptive thresholds ---
    wire [15:0] avg_block = (count_pix != 0)
                          ? (sum_block / count_pix)
                          : 16'd0;

    wire [15:0] avg_hf    = (count_pix != 0)
                          ? (sum_hf / count_pix)
                          : 16'd0;

    // --- per-pixel classification ---
    wire block_artifact = (edge_strength > (avg_block + 16'd20));
    wire hf_artifact    = (hf_strength   > (avg_hf    + 16'd25));

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            block_score   <= 0;
            hf_score      <= 0;
            sum_block     <= 0;
            sum_hf        <= 0;
            count_pix     <= 0;
            confidence    <= 0;
            fake_detected <= 0;
        end
        else if (enable) begin
            // update averages
            sum_block <= sum_block + edge_strength;
            sum_hf    <= sum_hf    + hf_strength;
            count_pix <= count_pix + 1;

            // accumulate scores
            if (block_artifact)
                block_score <= block_score + 1;

            if (hf_artifact)
                hf_score <= hf_score + 1;

            // confidence = weighted percentage
            // Total pixels = 1024 → percent ≈ score / 10
            confidence <= (block_score + hf_score) >> 3;

            // decision based on scores
            if ((block_score + hf_score) > 16'd180)
                fake_detected <= 1'b1;
            else
                fake_detected <= 1'b0;
        end
    end

endmodule
```
