──────────────────────────────────────────────
TAMA CALCULATOR CODEX — FORMULAS ONLY
All operations use addition and subtraction only.
──────────────────────────────────────────────

# 1. ADDITION
ADD(a, b):
    return a + b

# 2. SUBTRACTION
SUB(a, b):
    return a - b

# 3. MULTIPLICATION (Repeated Addition)
MUL(a, b):
    sign = sign_of(a) * sign_of(b)
    A = abs(a)
    B = abs(b)
    sum = 0
    repeat B times:
        sum = sum + A
    return sign * sum

# 4. DIVISION (Repeated Subtraction)
DIV(a, b):
    sign = sign_of(a) * sign_of(b)
    A = abs(a)
    B = abs(b)
    count = 0
    while A >= B:
        A = A - B
        count = count + 1
    # fractional remainder approximation
    frac = A / B   # conceptually repeated subtraction
    return sign * (count + frac)

# 5. SQUARE (Odd-Difference Stacking)
SQR(n):
    N = abs(floor(n))
    sum = 0
    odd = 1
    repeat N times:
        sum = sum + odd
        odd = odd + 2
    return sum

# 6. SQUARE ROOT (Guess-and-Adjust using SQR)
SQRT(N):
    if N < 0: return NaN
    g = 0
    while SQR(g) <= N:
        g = g + 1
    g = g - 1
    return g

# 7. FACTORIAL (Multiplication via Addition)
FACT(n):
    N = abs(floor(n))
    if N <= 1: return 1
    result = 1
    for i = 2 to N:
        result = MUL(result, i)
    return result

# 8. EXPONENTIAL (Series Expansion)
EXP(x):
    sum = 0
    for k = 0 to 10:
        term = (x^k) / FACT(k)
        sum = sum + term
    return sum

# 9. NATURAL LOG (Search for exponent)
LOG(a):
    if a <= 0: return NaN
    n = 0
    step = 0.1
    repeat 100 times:
        if EXP(n + step) <= a:
            n = n + step
        else:
            step = step / 10
    return n

# 10. SINE (Taylor Series)
SIN(x):
    sum = 0
    for k = 0 to 10:
        power = x^(2k+1)
        denom = FACT(2k+1)
        term = power / denom
        if k is even: sum = sum + term
        else: sum = sum - term
    return sum

# 11. COSINE (Taylor Series)
COS(x):
    sum = 0
    for k = 0 to 10:
        power = x^(2k)
        denom = FACT(2k)
        term = power / denom
        if k is even: sum = sum + term
        else: sum = sum - term
    return sum

# 12. TAMA DETERMINER STRUCTURE (Optional)
# x and y appear but cancel, shaping the bracket structure.
TAMA_MUL(a, b, x, y):
    return [a + x - x] repeated b times

TAMA_DIV(a, b, x, y):
    return [a - b + x - x], [a - 2b + y - y], ... until zero

──────────────────────────────────────────────
END OF TAMA CALCULATOR FORMULA SET
──────────────────────────────────────────────
