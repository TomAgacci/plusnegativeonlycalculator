──────────────────────────────────────────────
7‑8‑9 TAMA CPU — MICRO INSTRUCTION SET
──────────────────────────────────────────────

TRITS:
    7 = 0
    8 = 1
    9 = 2

WORD FORMAT (fixed width, conceptual):
    [ OPCODE | DEST | SRC | IMM ]

All fields are encoded in trinary using 7/8/9.

CORE REGISTERS:
    R7  = ACC   (accumulator)
    R8  = TMP   (temporary)
    R9  = FLAG  (status / carry / sign)

BASIC OPCODES (encoded as trits)
    ADD  = 7 8 7   ; ACC = ACC + SRC
    SUB  = 7 9 7   ; ACC = ACC - SRC
    MOV  = 8 7 7   ; DEST = SRC
    CLR  = 8 8 7   ; DEST = 0
    INC  = 8 9 7   ; DEST = DEST + 1
    DEC  = 9 7 7   ; DEST = DEST - 1
    CMP  = 9 8 7   ; FLAG = compare(ACC, SRC)
    JMP  = 9 9 7   ; PC = IMM
    JZ   = 7 7 8   ; if FLAG == 0 → PC = IMM
    JNZ  = 7 8 8   ; if FLAG != 0 → PC = IMM

IMMEDIATE ENCODING:
    Any small integer k is encoded as:
        k = 7 + (k - 7)(9 - 8)
    and stored as a trinary word of 7/8/9.

──────────────────────────────────────────────
MICROCODE MACROS (HIGH‑LEVEL OPS)
──────────────────────────────────────────────

MACRO: MUL_INT(A, B)
    ; ACC = A * B via repeated addition

    1) MOV R7, A        ; ACC = A
    2) MOV R8, B        ; TMP = B
    3) CLR R9           ; FLAG = 0 (used as counter)
LOOP_MUL:
    4) ADD R7, A        ; ACC = ACC + A
    5) DEC R8           ; TMP = TMP - 1
    6) CMP R8, 0        ; FLAG = (TMP == 0 ? 0 : 1)
    7) JNZ LOOP_MUL     ; repeat until TMP == 0

MACRO: DIV_INT(A, B)
    ; ACC = quotient, TMP = remainder

    1) MOV R7, A        ; ACC = A
    2) MOV R8, 0        ; TMP = 0 (quotient)
LOOP_DIV:
    3) CMP R7, B        ; FLAG = (ACC >= B ? 1 : 0)
    4) JZ END_DIV       ; if ACC < B → done
    5) SUB R7, B        ; ACC = ACC - B
    6) INC R8           ; TMP = TMP + 1
    7) JMP LOOP_DIV
END_DIV:
    ; R8 = quotient, R7 = remainder

MACRO: SQR_INT(N)
    ; ACC = N² via odd differences

    1) CLR R7           ; ACC = 0 (sum)
    2) MOV R8, 1        ; TMP = 1 (odd)
    3) MOV R9, N        ; FLAG = N (counter)
LOOP_SQR:
    4) ADD R7, R8       ; sum += odd
    5) ADD R8, 2        ; odd += 2
    6) DEC R9           ; counter--
    7) CMP R9, 0
    8) JNZ LOOP_SQR

MACRO: SQRT_INT(N)
    ; ACC ≈ √N via guess‑and‑adjust

    1) CLR R7           ; ACC = 0 (guess)
LOOP_SQRT:
    2) MOV R8, R7       ; TMP = guess
    3) CALL SQR_INT(R8) ; TMP = guess²
    4) CMP R8, N        ; FLAG = (guess² <= N ? 1 : 0)
    5) JZ DONE_SQRT     ; if guess² > N → done
    6) INC R7           ; guess++
    7) JMP LOOP_SQRT
DONE_SQRT:
    ; R7 = floor(√N)

──────────────────────────────────────────────
SCIENTIFIC KEYS → MICROCODE SEQUENCES
──────────────────────────────────────────────

KEY: SQR (x²)
    INPUT: x in ACC (R7)
    MICROCODE:
        1) MOV R9, R7       ; save x
        2) CALL SQR_INT(R9) ; ACC = x²

KEY: SQRT (√x)
    INPUT: x in ACC
    MICROCODE:
        1) MOV R9, R7       ; save x
        2) CALL SQRT_INT(R9); ACC ≈ √x

KEY: MUL (x × y)
    INPUT: x in ACC, y in TMP (R8)
    MICROCODE:
        1) MOV R9, R7       ; A = x
        2) MOV R7, R8       ; B = y
        3) CALL MUL_INT(R9, R7)
        ; ACC = x × y

KEY: DIV (x ÷ y)
    INPUT: x in ACC, y in TMP
    MICROCODE:
        1) MOV R9, R7       ; A = x
        2) MOV R7, R8       ; B = y
        3) CALL DIV_INT(R9, R7)
        ; R8 = quotient, R7 = remainder
        ; ACC can be set to quotient:
        4) MOV R7, R8

KEY: EXP (eˣ)  [approx via series]
    INPUT: x in ACC
    MICROCODE (conceptual):
        1) CLR R7           ; sum = 0
        2) MOV R8, 0        ; k = 0
LOOP_EXP:
        3) CALL TERM_EXP(x, k) ; term = x^k / k!
        4) ADD R7, term        ; sum += term
        5) INC R8              ; k++
        6) CMP R8, K_MAX
        7) JNZ LOOP_EXP
        ; ACC = sum

KEY: LOG (ln x) [search via EXP]
    INPUT: x in ACC
    MICROCODE (conceptual):
        1) CLR R7           ; n = 0
        2) MOV R8, STEP     ; step = small increment
LOOP_LOG:
        3) CALL EXP(n + step)
        4) CMP EXP_VAL, x
        5) JZ DONE_LOG      ; if exp > x → done
        6) ADD R7, STEP     ; n += step
        7) JMP LOOP_LOG
DONE_LOG:
        ; ACC = n

KEY: SIN (x)
    INPUT: x in ACC
    MICROCODE (conceptual):
        1) CLR R7           ; sum = 0
        2) MOV R8, 0        ; k = 0
LOOP_SIN:
        3) CALL TERM_SIN(x, k) ; term = (-1)^k x^(2k+1)/(2k+1)!
        4) ADD/SUB R7, term    ; sum += or -= term
        5) INC R8              ; k++
        6) CMP R8, K_MAX
        7) JNZ LOOP_SIN
        ; ACC = sum

KEY: COS (x)
    INPUT: x in ACC
    MICROCODE:
        same pattern as SIN, but term = (-1)^k x^(2k)/(2k)!

──────────────────────────────────────────────
7‑8‑9 ENCODING OF INSTRUCTIONS
──────────────────────────────────────────────

Each field (OPCODE, DEST, SRC, IMM) is encoded in trinary:

    Example:
        ADD R7, R8

    OPCODE = ADD = 7 8 7
    DEST   = R7  = 7 7 7
    SRC    = R8  = 7 8 8

    Full word:
        [ 7 8 7 | 7 7 7 | 7 8 8 | IMM ]

All bits/trits are 7/8/9 only.

──────────────────────────────────────────────
END OF 7‑8‑9 TAMA CPU MICROCODE SPEC
──────────────────────────────────────────────
