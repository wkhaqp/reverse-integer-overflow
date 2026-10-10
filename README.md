# reverse integer: why your LeetCode 7 answer breaks on 1534236469, and how to fix the overflow check in Python, Java, C++ and JavaScript

Most people who search for "reverse integer" already have working code. It passed `123`, passed `-123`, passed `120`. Then they hit submit, the judge fed it `1534236469`, and the answer came back `1056389759` instead of `0`.

That single test case is where the real problem lives. Reversing digits is a five-line loop. Deciding whether the result still fits in a signed 32-bit integer, *before* it stops fitting, is the actual exercise — and it's the part interviewers watch you write.

Here's what the problem asks, where the naive loop fails, and the version that passes in four languages.

## What the problem actually asks

> Given a signed 32-bit integer `x`, return `x` with its digits reversed. If reversing `x` causes the value to go outside the signed 32-bit range `[-2³¹, 2³¹ - 1]`, return `0` instead.

That range is `-2147483648` to `2147483647`. Beyond it, you return `0`, not a saturated value, not a bigger number, not an error.

| Input | Output | Why |
| --- | --- | --- |
| `123` | `321` | Straight reversal |
| `-123` | `-321` | Sign preserved |
| `120` | `21` | The trailing zero vanishes when it becomes a leading zero |
| `1534236469` | `0` | Reversed is `9646324351`, which overflows |
| `-2147483648` | `0` | Reversed is `8463847412`, which overflows |
| `1463847412` | `2147483641` | Reversed is just inside the limit |

Note the last two rows. One is the smallest legal input and returns `0`; the other reverses to a number one digit away from the ceiling and returns successfully. The boundary is not decorative.

## The solution that looks correct

Almost everyone writes this first:

java
public int reverse(int x) {
    int res = 0;
    while (x != 0) {
        res = res * 10 + x % 10;
        x /= 10;
    }
    return res;
}


Trace it on `123`: `res` becomes `3`, then `32`, then `321`. On `-123` Java's `%` gives `-3`, so it lands on `-321`. It even handles `120` correctly, because the first extracted digit is `0` and `0 * 10 + 0` is still `0`.

Then it meets `1534236469`. The digits come out of the loop fine, but the intermediate `964632435` gets multiplied by 10, the product exceeds `2147483647`, and Java quietly rolls it over to `1056389759`. No exception. No warning. A wrong answer with a straight face.

### Why the obvious fix doesn't fire

The reflex is to append a sanity check:

java
res = res * 10 + x % 10;
if (res > Integer.MAX_VALUE || res < Integer.MIN_VALUE) return 0;


This is dead code. Once `res` is declared as an `int`, it can never hold a value larger than `Integer.MAX_VALUE`. IntelliJ will grey it out with "condition is always false." The overflow already happened on the line above; the check is inspecting the wreckage.

The same trap exists in C, C++, C#, Go and Rust's release builds. In C++ it's worse than a wrong answer: signed integer overflow is undefined behavior, so the compiler is entitled to assume it never happens and optimize your guard away entirely.

| Language | Behavior on `int` overflow | `-123 % 10` | `-123 / 10` |
| --- | --- | --- | --- |
| Java | Silent wraparound | `-3` | `-12` |
| C / C++ | Undefined behavior | `-3` | `-12` |
| # (unchecked) | Silent wraparound | `-3` | `-12` |
| JavaScript | No overflow, but `/` is float division | `-3` | `-12.3` |
| Python | No overflow at all | `7` | `-13` |

## The check that works: guard the multiplication, not the result

The ceiling is `2147483647`. Before multiplying by 10, you need to know whether `res * 10` can still fit. Since `2147483647 / 10` is `214748364` with a remainder of `7`, the safe rule is:

- if `res > 214748364`, the next multiplication overflows, so return `0`;
- if `res < -214748364`, same thing on the negative side.

You can write the full textbook condition with the remainder comparison attached (`res == 214748364 && digit > 7`). Many published solutions do. It's harmless, but for a valid 32-bit input the equality branch is unreachable — reaching `res == 214748364` requires a ten-digit original, and the largest legal input starts with `2`. Plain inequality is enough.

**Java**

java
class Solution {
    public int reverse(int x) {
        int res = 0;
        while (x != 0) {
            if (res > 214748364 || res < -214748364) return 0;
            res = res * 10 + x % 10;
            x /= 10;
        }
        return res;
    }
}


**C++**

cpp
class Solution {
public:
    int reverse(int x) {
        int res = 0;
        while (x != 0) {
            if (res > 214748364 || res < -214748364) return 0;
            res = res * 10 + x % 10;
            x /= 10;
        }
        return res;
    }
};


**JavaScript** — the divisor is the part people miss. `-123 / 10` is `-12.3`, not `-12`, so `x` drifts into fractional territory and the loop misbehaves. Use `Math.trunc`, or subtract the digit first:

js
var reverse = function(x) {
    let res = 0;
    while (x !== 0) {
        const digit = x % 10;
        if (res > 214748364 || res < -214748364) return 0;
        res = res * 10 + digit;
        x = (x - digit) / 10;   // exact integer division here
    }
    return res;
};


**Python** — this one needs more than a copy-paste, and it's the topic of the next section.

## The Python trap that hangs your loop

Python integers are arbitrary precision, so nothing rolls over and you have to enforce the bound yourself. That part is easy. The hard part is that Python's `%` and `//` don't match Java's.

`-123 % 10` in Python is `7`, not `-3`. And `-1 // 10` is `-1`, because floor division rounds toward negative infinity. Drop the Java loop into Python unchanged and the last iteration does this:

python
x = -1
x // 10   # -1, not 0


`x` never reaches zero. The loop runs forever.

The fix is to pull each digit toward zero before using it, then advance `x` by subtracting that digit:

python
class Solution:
    def reverse(self, x: int) -> int:
        res = 0
        while x != 0:
            digit = int(x % 10)
            if x < 0 and digit > 0:
                digit -= 10          # -123 % 10 -> 7, we want -3
            if res > 214748364 or res < -214748364:
                return 0
            res = res * 10 + digit
            x = (x - digit) // 10     # now this is real integer division
        return res


One more Python footnote worth knowing: don't write the bound as `-(2**31) // 10`. Python floors `-2147483648 // 10` to `-214748365`, which is one off from the `-214748364` you actually need. Integer literals sidestep the whole question.

AlgoMonster's write-up of this problem walks through exactly that negative-number adjustment step by step, `/ -123` traced iteration by iteration with the digit corrected from `7` to `-3` each pass, and the overflow guard expressed as a range check against `[-214748364, 214748364]`. It's laid out as prose plus diagrams rather than a code dump, which is handy when the modulo behavior is the thing confusing you. The walkthrough is public: [👉 read the full Reverse Integer walkthrough on AlgoMonster](https://algo.monster/liteproblems/7?fpr=li69).

## String reversal vs digit math

You can also do it with strings: convert to text, reverse, parse back, range-check, return `0` if out of bounds. In Python and JavaScript that's three short lines and it works.

It's a fine first answer in an interview, with two caveats. The problem statement assumes you can't store 64-bit integers, and the string route usually leans on a wider type or a parser to do its range check. And the interviewer is generally testing whether you can do arithmetic defensively, which the string answer skips. Say it out loud as a stepping stone, then write the digit loop.

Complexity is the same either way: **O(log₁₀|x|)** time, since you process each digit once, and **O(1)** space. A 10-digit number is 10 iterations. There's nothing to optimize here, which is why the overflow logic is the whole grade.

## The checklist an interviewer is actually scoring

- Negative input: sign preserved, no infinite loop.
- Trailing zeros: `120` must give `21`, not `021`.
- `x == 0`: returns `0`, loop body never runs.
- The overflow guard sits **before** the multiply, not after.
- You can state the boundary as `-2147483648` and `2147483647` from memory.
- You don't reach for `long` when the prompt forbids 64-bit storage.
- You know the answer for `1534236469` is `0` because `9646324351 > 2147483647`.

Test `1534236469` and `-2147483648` before you submit anything. Those two catch the failure modes.

## Where this question sits in the bigger picture

Reverse Integer is filed under math and digit manipulation, but the skill it exercises shows up everywhere: extract a digit, mutate an accumulator, verify a bound before committing. Add Digits, Palindrome Number, String to Integer (atoi), and Multiply Strings all reward the same instinct.

The reason people get stuck isn't the loop. It's that they treat the problem as "reverse a number" instead of "reverse a number and prove the result is representable." That gap is a pattern-recognition gap, and it's the thing AlgoMonster is built to close. The platform's pitch is straightforward: it's a text-and-diagram curriculum organized by reusable coding patterns instead of a bank of 3,000 problems with no map. Its own copy says the content is written by Google engineers and competitive programmers, and the paid tier advertises 325 lessons and problems, 678 illustrations and 3,000+ editorials, plus unlimited use of an AI assistant and Speedrun mode for timed review.

Worth knowing before you buy:

- **It's text and diagrams, no video.** Reviewers consistently flag this as the main mismatch. If you learn from someone talking through a whiteboard, this format will frustrate you.
- **System design coverage is introductory.** It's fine for a first pass, not enough for a senior loop with a dedicated design round.
- **No team or enterprise plan.** It's built for one candidate prepping alone.

If your actual problem is "I can write the code but I keep missing the edge cases and the pattern behind them," that's the exact thing it addresses. [👉 browse the pattern library and the free problem editorials](https://algo.monster?fpr=li69).

## AlgoMonster plans and pricing

AlgoMonster doesn't do monthly billing. The subscribe page currently lists a free tier, a yearly plan, and a one-time lifetime plan:

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and problems | $0 | — | [ Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; 325 lessons and problems, 678 illustrations, 3,000+ editorials; 12 months of access | $8.25/mo, billed as $99/year. Subscribe page lists a 3-day free trial before the first charge | Annual | [ Check the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in yearly, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 | One-time | [ See the lifetime plan](https://algo.monster/subscribe?fpr=li69) |

Two honest caveats about those numbers.

AlgoMonster rotates promotions constantly, and the pricing on that page has moved around a lot historically. The official community forum has threads from users asking for the old `$99` lifetime sale to come back, which tells you the floor has been lower than the list price more than once. Third-party reviews quote figures from `$89/year` to `$459 lifetime` depending on when they were written. Treat the checkout total as the only real number, and check it before you commit.

The second caveat: lesson and illustration counts on the subscribe page change as they publish new content. If the exact count matters to you, look at the page, not at a blog post — including this one.

## Should you pay for it?

For a one-time push with an interview in six weeks, the yearly plan at `$99` is the sensible buy, and it's comparable to a single month of most live-coaching options. For someone who interviews every couple of years, the `$189` lifetime sits below two years of the annual plan and is the better arithmetic.

If you already have a method and only need volume, LeetCode's problem bank is larger and cheaper per problem, and no pattern course replaces reps. If your gap is video explanations, this isn't the platform for you.

What you're buying here is structure for the class of problem that just cost you a submission — the guard before the multiply, the modulo sign, the `-1 // 10` that never terminates. Those are cheap to learn and expensive to rediscover at 30 minutes into a 45-minute screen. [👉 Check the current AlgoMonster pricing and start the free tier](https://algo.monster/subscribe?fpr=li69).

## Quick answers

**Why does `1534236469` return `0`?** Its reversal is `9646324351`, which exceeds `2147483647`, so the problem requires `0`.

**Can I just use `long` or a 64-bit int?** The prompt says assume you can't store outside the 32-bit range. Some languages let you cheat; the interviewer usually notices.

**Why does my Python loop never finish on negative input?** Because `-1 // 10 == -1`. Python floors toward negative infinity and its `%` returns a positive digit for negative `x`. Correct the digit, then advance `x` with `(x - digit) // 10`.

**Is O(1) space achievable?** Yes. Five variables, one pass over the digits.

**Does the equality case `res == 214748364 && digit > 7` ever trigger?** Not for a legal 32-bit input. Keep the simple inequality check unless you find the fuller condition clearer to read.
