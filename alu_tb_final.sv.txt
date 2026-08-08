// =============================================================================
// Module      : alu_tb
// Description : Self-checking testbench for the 8-bit ALU.
//                 - Directed tests: one per opcode + a few known edge cases
//                 - Constrained-random tests checked against a reference model
//                 - Immediate SystemVerilog Assertions (SVA) on flag behavior
//                 - Functional coverage on opcode and flags
//
// Note: The ALU itself has no clock (it's pure combinational logic), so this
// testbench has no clock either. Each test simply: drives inputs, waits a
// tiny delay for the logic to settle, then checks the outputs. This is the
// standard way to test a combinational block.
//
// Run (Icarus Verilog):
//     iverilog -g2012 -o sim alu.sv alu_tb.sv
//     vvp sim
//
// Author      : Luqman Shariff
// =============================================================================
`timescale 1ns/1ps

module alu_tb;

    localparam logic [2:0] OP_ADD = 3'b000;
    localparam logic [2:0] OP_SUB = 3'b001;
    localparam logic [2:0] OP_AND = 3'b010;
    localparam logic [2:0] OP_OR  = 3'b011;
    localparam logic [2:0] OP_XOR = 3'b100;
    localparam logic [2:0] OP_NOT = 3'b101;
    localparam logic [2:0] OP_SHL = 3'b110;
    localparam logic [2:0] OP_SHR = 3'b111;

    // -------------------------------------------------------------------
    // Signals connected to the DUT (Device Under Test)
    // -------------------------------------------------------------------
    logic [7:0] a, b;
    logic [2:0] opcode;
    logic [7:0] result;
    logic       zero, carry_out, overflow;

    alu dut (
        .a(a), .b(b), .opcode(opcode),
        .result(result), .zero(zero),
        .carry_out(carry_out), .overflow(overflow)
    );

    // -------------------------------------------------------------------
    // Reference model: computes the "correct" answer independently of the
    // DUT, using plain SystemVerilog operators. The testbench compares the
    // DUT's result against this every time.
    // -------------------------------------------------------------------
    function automatic logic [7:0] ref_result(
        input logic [7:0] a_i,
        input logic [7:0] b_i,
        input logic [2:0] op_i
    );
        case (op_i)
            OP_ADD:  ref_result = a_i + b_i;
            OP_SUB:  ref_result = a_i - b_i;
            OP_AND:  ref_result = a_i & b_i;
            OP_OR:   ref_result = a_i | b_i;
            OP_XOR:  ref_result = a_i ^ b_i;
            OP_NOT:  ref_result = ~a_i;
            OP_SHL:  ref_result = a_i << 1;
            OP_SHR:  ref_result = a_i >> 1;
            default: ref_result = 8'b0;
        endcase
    endfunction

    int test_count  = 0;
    int error_count = 0;

    // -------------------------------------------------------------------
    // Functional coverage - tracked manually with plain arrays instead of
    // a SystemVerilog "covergroup". because icarus does not support covergroup
    // construct
    // -------------------------------------------------------------------
    bit [7:0] opcode_seen;   // one bit per opcode (0 to 7), set when hit
    bit       zero_seen_0, zero_seen_1;
    bit       carry_seen_0, carry_seen_1;
    bit       overflow_seen_0, overflow_seen_1;

    task automatic record_coverage();
        opcode_seen[opcode] = 1'b1;

        if (zero) zero_seen_1 = 1'b1; else zero_seen_0 = 1'b1;
        if (carry_out) carry_seen_1 = 1'b1; else carry_seen_0 = 1'b1;
        if (overflow) overflow_seen_1 = 1'b1; else overflow_seen_0 = 1'b1;
    endtask

   //-------------------------------------------------------------------
    // apply(): runs one test case - drives inputs, lets the logic settle,
    // then checks the result and the flags. Tests are numbered (tc_id)
    // instead of named, since Icarus doesn't play nice with strings as
    // task arguments.
    // -------------------------------------------------------------------
    task automatic apply(input logic [7:0] a_i,
                          input logic [7:0] b_i,
                          input logic [2:0] op_i,
                          input int         tc_id);
        a      = a_i;
        b      = b_i;
        opcode = op_i;
        #1; // let the combinational logic settle before checking anything

        test_count++;

        // ---- Check 1: result correctness (scoreboard check) ----
        if (result !== ref_result(a, b, opcode)) begin
            error_count++;
            $display("FAIL [test %0d] a=%0d b=%0d op=%0b  expected=%0d got=%0d",
                      tc_id, a, b, opcode, ref_result(a, b, opcode), result);
        end

        // ---- Check 2: SVA - zero flag must always match result==0 ----
        // An immediate assertion checks a condition right now, at this
        // point in the simulation, instead of waiting for a clock edge.
        assert (zero == (result == 8'b0)) else begin
            error_count++;
            $error("[SVA] zero flag wrong at test %0d: result=%0d zero=%0b", tc_id, result, zero);
        end

        // ---- Check 3: SVA - NOT should never raise carry_out/overflow ----
        if (opcode == OP_NOT) begin
            assert (carry_out == 1'b0 && overflow == 1'b0) else begin
                error_count++;
                $error("[SVA] NOT incorrectly set carry_out/overflow at test %0d", tc_id);
            end
        end

        // ---- Functional coverage sample ----
        record_coverage();
    endtask

    initial begin
        $display("=== 8-bit ALU Verification Start ===");

        // ---- Directed tests: one per opcode, plus a few edge cases ----
        // (see comments for what each one is checking)
        apply(8'd5,   8'd3,   OP_ADD, 1);  // ADD basic
        apply(8'd255, 8'd1,   OP_ADD, 2);  // ADD carry, no signed overflow
        apply(8'd128, 8'd128, OP_ADD, 3);  // ADD signed overflow
        apply(8'd0,   8'd0,   OP_ADD, 4);  // ADD zero result

        apply(8'd5,   8'd3,   OP_SUB, 5);  // SUB basic
        apply(8'd3,   8'd5,   OP_SUB, 6);  // SUB borrow
        apply(8'd128, 8'd1,   OP_SUB, 7);  // SUB signed overflow
        apply(8'd7,   8'd7,   OP_SUB, 8);  // SUB zero result

        apply(8'hF0,  8'h0F,  OP_AND, 9);   // AND disjoint bits
        apply(8'h00,  8'hFF,  OP_OR,  10);  // OR basic
        apply(8'hAA,  8'h55,  OP_XOR, 11);  // XOR alternating
        apply(8'hAA,  8'hAA,  OP_XOR, 12);  // XOR self -> zero

        apply(8'h0F,  8'h00,  OP_NOT, 13);  // NOT basic
        apply(8'h00,  8'h00,  OP_NOT, 14);  // NOT zero -> all ones

        apply(8'h01,  8'd0,   OP_SHL, 15);  // SHL basic
        apply(8'h80,  8'd0,   OP_SHR, 16);  // SHR basic

        // ---- Constrained-random regression: 50 random test cases ----
        for (int i = 0; i < 50; i++) begin
            apply($urandom_range(0, 255),
                  $urandom_range(0, 255),
                  $urandom_range(0, 7),
                  17 + i);
        end

        // ---- Final report ----
        $display("=== 8-bit ALU Verification Complete ===");
        $display("Tests run   : %0d", test_count);
        $display("Errors      : %0d", error_count);

        // Manual coverage summary: how many of the 8 opcodes did we hit,
        // and did we ever see each flag in both its 0 and 1 states?
        begin
            int opcodes_hit;
            int total_bins;
            int bins_hit;

            opcodes_hit = 0;
            for (int op_i = 0; op_i < 8; op_i++)
                if (opcode_seen[op_i]) opcodes_hit++;

            // 8 opcode bins + 6 flag-state bins (zero/carry/overflow, each 0 and 1)
            total_bins = 8 + 6;
            bins_hit   = opcodes_hit
                       + zero_seen_0     + zero_seen_1
                       + carry_seen_0    + carry_seen_1
                       + overflow_seen_0 + overflow_seen_1;

            $display("Opcode coverage : %0d / 8 opcodes exercised", opcodes_hit);
            $display("Flag coverage   : zero(0/1)=%0d/%0d  carry(0/1)=%0d/%0d  overflow(0/1)=%0d/%0d",
                      zero_seen_0, zero_seen_1, carry_seen_0, carry_seen_1,
                      overflow_seen_0, overflow_seen_1);
            $display("Overall functional coverage : %0.2f%% (%0d / %0d bins)",
                      (100.0 * bins_hit) / total_bins, bins_hit, total_bins);
        end

        if (error_count == 0)
            $display("RESULT: ALL TESTS PASSED");
        else
            $display("RESULT: %0d TEST(S) FAILED", error_count);

        $finish;
    end

endmodule
