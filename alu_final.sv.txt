// =============================================================================
// Module      : alu
// Description : 8-bit Arithmetic Logic Unit (ALU).
//               Supports 8 operations, selected using a 3-bit opcode:
//                 ADD, SUB, AND, OR, XOR, NOT, shift-left by 1, shift-right by 1
//
// Flags       : zero      - asserted when result == 0
//               carry_out - carry out of ADD / borrow out of SUB
//               overflow  - signed 2's-complement overflow (ADD/SUB only)
//
// Author      : Luqman Shariff
// =============================================================================

module alu (
    input  logic [7:0] a,
    input  logic [7:0] b,
    input  logic [2:0] opcode,

    output logic [7:0] result,
    output logic       zero,
    output logic       carry_out,
    output logic       overflow
);

    // ---------------------------------------------------------------------
    // Opcode encoding - one value per supported operation
    // ---------------------------------------------------------------------
    localparam logic [2:0] OP_ADD = 3'b000;
    localparam logic [2:0] OP_SUB = 3'b001;
    localparam logic [2:0] OP_AND = 3'b010;
    localparam logic [2:0] OP_OR  = 3'b011;
    localparam logic [2:0] OP_XOR = 3'b100;
    localparam logic [2:0] OP_NOT = 3'b101;
    localparam logic [2:0] OP_SHL = 3'b110;
    localparam logic [2:0] OP_SHR = 3'b111;

    // 9-bit wide sums so the extra bit can hold the carry / borrow
    logic [8:0] add_ext;
    logic [8:0] sub_ext;

    always_comb begin
        add_ext = {1'b0, a} + {1'b0, b};
        sub_ext = {1'b0, a} - {1'b0, b};

        // Default every signal at the top of the block. Without this,
        // a case branch that forgets to set carry_out/overflow would
        // infer an unwanted latch instead of a clean combinational circuit.
        result    = 8'b0;
        carry_out = 1'b0;
        overflow  = 1'b0;

        case (opcode)
            OP_ADD: begin
                result    = add_ext[7:0];
                carry_out = add_ext[8];
                // Signed overflow rule for addition: overflow can only
                // happen if both operands have the SAME sign bit, and the
                // result ends up with a DIFFERENT sign bit.
                overflow  = (a[7] == b[7]) && (result[7] != a[7]);
            end

            OP_SUB: begin
                result    = sub_ext[7:0];
                carry_out = sub_ext[8];  // 1 means a < b (a "borrow" happened)
                // Signed overflow rule for subtraction: overflow can only
                // happen if the operands have DIFFERENT sign bits, and the
                // result's sign bit doesn't match a's sign bit.
                overflow  = (a[7] != b[7]) && (result[7] != a[7]);
            end

            OP_AND:  result = a & b;
            OP_OR:   result = a | b;
            OP_XOR:  result = a ^ b;
            OP_NOT:  result = ~a;

            OP_SHL:  result = a << 1;   // shift left by 1, fill with 0
            OP_SHR:  result = a >> 1;   // shift right by 1, fill with 0

            default: result = 8'b0;
        endcase
    end

    // zero flag is simple: true whenever the result happens to be 0
    assign zero = (result == 8'b0);

endmodule
