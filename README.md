================================================================================
HW5 -- Fault Attack on AES-128 (Differential Fault Analysis)
README -- Lines Changed
================================================================================

OVERVIEW
--------
This submission implements a Differential Fault Analysis (DFA) attack on the
tiny-AES-c AES-128 implementation to recover the round-10 subkey (K10).
The attack is based on: "DFA on AES" by Christophe Giraud.

Three existing files were modified. No new files were added.

================================================================================
FILE 1: aes.c
================================================================================

CHANGE 1 -- Lines 41-52 (after the #include block)
  Added four global variables that control fault injection:

    uint8_t g_fault_enable  = 0;  -- Set to 1 to enable fault injection.
    uint8_t g_fault_byte    = 0;  -- Flat byte index (0-15) into the M9 state.
    uint8_t g_fault_bit     = 0;  -- Bit position (0-7) to flip in that byte.
    uint8_t g_m9_snapshot[16];    -- Captures M9 BEFORE the fault is applied.

  These globals let test.c control when and where a fault is injected without
  modifying the AES encryption function signature.

CHANGE 2 -- Lines 447-463 (inside Cipher(), after AddRoundKey in round 9)
  Added a fault injection block that executes immediately after
  AddRoundKey(round, state, RoundKey) when round == Nr-1 (== 9 for AES-128):

    if (round == (uint8_t)(Nr - 1))
    {
      memcpy(g_m9_snapshot, (uint8_t*)state, AES_BLOCKLEN);
      if (g_fault_enable)
      {
        ((uint8_t*)state)[g_fault_byte] ^= (uint8_t)(1u << g_fault_bit);
      }
    }

  At this point in Cipher(), the state IS M9 -- the output of round 9 and the
  input to the final round (round 10). The memcpy captures M9 for reference;
  the XOR flips one bit to emulate a hardware glitch.

================================================================================
FILE 2: aes.h
================================================================================

CHANGE 3 -- Lines 91-100 (just before the final #endif)
  Added extern declarations for the four globals defined in aes.c so that
  test.c (and any other translation unit) can reference them:

    extern uint8_t g_fault_enable;
    extern uint8_t g_fault_byte;
    extern uint8_t g_fault_bit;
    extern uint8_t g_m9_snapshot[16];

================================================================================
FILE 3: test.c
================================================================================

CHANGE 4 -- Line 22 (forward declaration block)
  Added forward declaration for the new DFA test function:

    static void test_fault_attack_dfa(void);

CHANGE 5 -- Lines 45-48 (inside main(), after test_encrypt_ecb_verbose())
  Added a call to run the DFA attack as part of the test suite:

    test_fault_attack_dfa();

CHANGE 6 -- Lines 321-514 (appended after the last original function)
  Added the full test_fault_attack_dfa() function implementing the DFA attack.
  Key sections within it:

    Lines 321-360  -- Block comment documenting the algorithm and assignment
                      step references.
    Lines 362-394  -- Local copy of the AES S-box (fa_sbox), needed because
                      the original sbox in aes.c is declared static.
    Lines 396-407  -- Setup: known key, plaintext, AES context init.
    Lines 409-418  -- Fault-free encryption to obtain reference ciphertext C
                      (Assignment Step 2).
    Lines 420-432  -- Outer loop: iterates over all 16 ciphertext bytes (ob).
    Lines 434-437  -- ShiftRows inverse mapping: computes which M9 byte (mb)
                      feeds ciphertext byte ob.
    Lines 439-441  -- Initialize all 256 values as candidates for M9[mb]
                      (Assignment Step 3).
    Lines 447-480  -- Inner loop (bits 0-7): injects fault (Step 1), records
                      C' (Step 2), and eliminates inconsistent candidates
                      (Step 4). Intersects across multiple faults (Step 5).
    Lines 482-489  -- Finds the single remaining candidate = M9[mb] (Step 5).
    Lines 491-494  -- Stores M9 byte and reconstructs K10[ob] (Steps 6-7):
                        K10[ob] = sbox[M9[mb]] XOR C[ob]
    Lines 497-509  -- Prints recovered K10 and verifies against the actual
                      key schedule (ctx.RoundKey[160..175]).

================================================================================
HOW TO BUILD AND RUN
================================================================================
Using WSL (or any Linux/macOS environment with gcc):

  gcc -Wall -Os -DECB=1 -c -o aes.o aes.c
  gcc -Wall -Os -c -o test.o test.c
  gcc -Wall -Os -o test.elf aes.o test.o
  ./test.elf

Using make:
  make

Expected output (at the end of the test run):
  Fault Attack DFA: SUCCESS -- K10 recovered correctly!
  Recovered K10:  d0 14 f9 a8 c9 ee 25 89 e1 3f 0c c8 b6 63 0c a6
  Actual    K10:  d0 14 f9 a8 c9 ee 25 89 e1 3f 0c c8 b6 63 0c a6

================================================================================
