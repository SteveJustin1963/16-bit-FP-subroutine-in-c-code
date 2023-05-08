# 16-bit-FP-subroutine-in-c-code

```
typedef struct float16 {
    uint16_t sign : 1;
    uint16_t exponent : 5;
    uint16_t mantissa : 10;
} float16;

typedef union float16_union {
    float16 value;
    uint16_t bits;
} float16_union;

float16 float16_add(float16 a, float16 b) {
    float16_union ua, ub, ur;
    ua.value = a;
    ub.value = b;

    int32_t am = ua.value.mantissa, bm = ub.value.mantissa;
    int32_t ae = ua.value.exponent, be = ub.value.exponent;
    uint32_t sign_ab = ua.value.sign ^ ub.value.sign;

    // handle special cases
    if (ae == 31) {
        if (am) {
            ur.bits = 0x7FFF;  // propagate canonical NaN
        }
        return a;  // a is infinity or NaN
    }
    if (be == 31) {
        if (bm) {
            ur.bits = 0x7FFF;  // propagate canonical NaN
        }
        return b;  // b is infinity or NaN
    }
    if (ae == 0 && am == 0) return b;  // a is zero
    if (be == 0 && bm == 0) return a;  // b is zero

    // align exponents
    int32_t diff = ae - be;
    if (diff > 0) {
        bm = (bm >> diff) | ((bm & ((1 << diff) - 1)) << (10 - diff));
        be = ae;
    } else if (diff < 0) {
        am = (am >> (-diff)) | ((am & ((1 << (-diff)) - 1)) << (10 + diff));
        ae = be;
    }

    // handle subtraction when signs are different
    if (sign_ab && am > bm) {
        int32_t tmp = am;
        am = bm;
        bm = tmp;
    }

    // perform addition or subtraction
    int32_t mantissa = am + (sign_ab ? -bm : bm);

    // handle subnormal numbers
    if (ae == 0) {
        int32_t shift = -14 - ae;
        mantissa <<= shift;
        ae += shift;
    }

    // handle rounding (round-to-nearest-ties-to-even)
    if ((mantissa & 0x400) && ((mantissa & 0x3FF) || (mantissa & 0x800))) {
        mantissa += 0x400;
        if ((mantissa & 0x7FF) == 0) {
            mantissa >>= 1;
            ae++;
        }
    }

    // handle overflow and underflow
    if (ae >= 31) {
        ur.value.sign = sign_ab;
        ur.value.exponent = 31;
        ur.value.mantissa = 0;
    } else if (ae <= 0) {
        int32_t shift = ae - 1;
        if (shift < -10) {
            ur.value.sign = sign_ab;
            ur.value.exponent = 0;
            ur.value.mantissa = 0;
        } else {
            mantissa = (mantissa >> (-shift)) | ((mantissa & ((1 << (-shift)) - 1)) != 0);
            ur.value.sign = sign_ab;
            ur.value.exponent = 0;
                        ur.value.mantissa = mantissa;
        }
    } else {
        ur.value.sign = sign_ab;
        ur.value.exponent = ae;
        ur.value.mantissa = mantissa & 0x3FF;
    }

    return ur.value;
}
```


