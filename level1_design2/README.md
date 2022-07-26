# Sequence Detector Verification

![Screenshot from 2022-07-30 11-44-34](https://user-images.githubusercontent.com/41594627/182002996-7ace628d-8bdd-4560-8181-6833fb666db4.png)
**Fig.1 A screenshot showing my Gitpod id**

## Verification Strategy
For this test, the device under test (DUT) is a sequence detector which was implemented as a finite State Machine (FSM). The technique employed in this test was Equivalence Check. Equivalence check involves checking a property of the DUT against an equivalent property of the "golden model". The property of the golden model is usually abstracted as a boolean (or logic) or mathematical function.
According to theory, two FSMs (especially Moore Machines) represented as black boxes are equivalent if:
- for every finite input sequence, they produce identical output sequence
- in addition, for Mealy Machine, they must transition to equivalent states for the same finite input sequence.

The states of the "golden model" were encoded as a 3-bit states, then logic functions were derived for the next state transition and output.

### Verification Environment
This test is a [CoCoTb](https://www.cocotb.org/) Python based test.

- After importing the relevant libraries, both the DUT were initialised to a known state (IDLE STATE)

```
# reset
  dut.reset.value = 1
  s2, s1, s0 = 0, 0, 0      #initialise the current state of the golden model to IDLE state
  await FallingEdge(dut.clk)  
  dut.reset.value = 0
  await FallingEdge(dut.clk)
```

-  A while loop was constructed to run for a long but finite time (say 100 times). This is because with many input sequence the bug will be easily captured.
- In this loop a 1-bit random number is being generated for every cycle. This 1-bit number is asigned to the inputs of the DUT and "golden model"

```
seq_inp = random.randint(0,1)      #generate a 1-bit random number
        
print("bit sequence:", seq_inp)
dut.inp_bit.value = seq_inp       #feed in the generated number to the input of the DUT
x = seq_inp                       #also assign the number to the input of the "golden" FSM model
```

- The next state logic of the "golden model" is being computed for every cycle. This is the next state logic function (recall it's a 3-bit encoded state, so a function derived for each bit):


```
#the next state logic of the golden model:
#given the current state and input
#compute the next state of the golden model
N2 = s1&s0&x
N1 = (s0&~x) | (s1&~s0&x)
N0 = (~s2&~s0&x) | (~s2&~s1&x) | (~s1&s0&x)
```

- The new curent states of the DUT and golden model are being "asserted" to check if they transitioned to equivalent states. Their outputs were also commpared.

```
#the output logic of the golden model:
output = (s2&~s0) | (s2&~s1)

#check if the current state of the golden model and DUT are equivalent
#and if their outputs are identical
assert dut_current_state[2] == s0,   "the DUT and the golden model did not transition to equivalent state"
assert dut_current_state[1] == s1,   "the DUT and the golden model did not transition to equivalent state"
assert dut_current_state[0] == s2,   "the DUT and the golden model did not transition to equivalent state"
assert dut.seq_seen.value == output, "incorrect output sequence"
cycle = cycle + 1 
await FallingEdge(dut.clk)
```

## Test Scenerio
1. Initial state: IDLE, encoded as '000'
- input bit: 0   expected state: 000  observed state: 000
- input bit: 1   expected state: 001  observed state: 001
- input bit: 1   expected state: 001 observed state:  000
```
42500.00ns INFO     test_seq_bug1 failed
               Traceback (most recent call last):
                 File "/workspace/challenges-Emmanuel-Innocent/level1_design2/test_seq_detect_1011.py", line 64, in test_seq_bug1
                   assert dut_current_state[2] == s0,   "the DUT and the golden model did not transition to equivalent state"
               AssertionError: the DUT and the golden model did not transition to equivalent state
```

2. Initial state: IDLE, encoded as '000'
- input bit: 0   expected state: 000  observed state: 000
- input bit: 0   expected state: 000  observed state: 000
- input bit: 0   expected state: 000  observed state: 000
- input bit: 0   expected state: 000  observed state: 000
- input bit: 1   expected state: 001  observed state: 001
- input bit: 0   expected state: 010  observed state: 010
- input bit: 1   expected state: 011  observed state: 011
- input bit: 1   expected state: 100  observed state: 100
- input bit: 0   expected state: 000  observed state: 000
- input bit: 1   expected state: 001  observed state: 001
- input bit: 1   expected state: 001  observed state: 000

```
122500.00ns INFO     test_seq_bug1 failed
                     Traceback (most recent call last):
                       File "/workspace/challenges-Emmanuel-Innocent/level1_design2/test_seq_detect_1011.py", line 64, in test_seq_bug1
                         assert dut_current_state[2] == s0,   "the DUT and the golden model did not transition to equivalent state"
                     AssertionError: the DUT and the golden model did not transition to equivalent state
```

3. Initial state: IDLE, encoded as '000'
- input bit: 0   expected state: 000  observed state: 000
- input bit: 0   expected state: 000  observed state: 000
- input bit: 0   expected state: 000  observed state: 000
- input bit: 0   expected state: 000  observed state: 000
- input bit: 1   expected state: 001  observed state: 001
- input bit: 1   expected state: 001  observed state: 001
- input bit: 1   expected state: 001  observed state: 001
- input bit: 0   expected state: 010  observed state: 010
- input bit: 0   expected state: 000  observed state: 000
- input bit: 1   expected state: 001  observed state: 001
- input bit: 1   expected state: 001  observed state: 001
- input bit: 0   expected state: 010  observed state: 010
- input bit: 1   expected state: 011  observed state: 011
- input bit: 0   expected state: 010  observed state: 000

```
152500.00ns INFO     test_seq_bug1 failed
                     Traceback (most recent call last):
                       File "/workspace/challenges-Emmanuel-Innocent/level1_design2/test_seq_detect_1011.py", line 65, in test_seq_bug1
                         assert dut_current_state[1] == s1,   "the DUT and the golden model did not transition to equivalent state"
                     AssertionError: the DUT and the golden model did not transition to equivalent state
```
From the three results above, it is observed that an **assertion error** is usually raised when:
- the previous state is SEQ_1 (encoded as '001') and the next input bit is '1'
   - with that given previous state and input, the golden model transitions to state SEQ_1(encoded as '001'), that is, it remains in the same state. This is valid because of overlapping(as defined in the design).
   - However, for the same previous state and input, the DUT transitions to state IDLE(encoded as '000'). This is incorrect as overlapping would not be taken care of in the design.
- the previous state is SEQ_101 (encoded as '011') and the next input bit is '0'
  - with this given condition the golden model transitions to state SEQ_10(encoded as '010'). This is valid because of overlapping(as defined in the design).
   - However, for the same previous state and input, the DUT transitions to state IDLE (encoded as '000'). This is incorrect as overlapping would not be taken care of in the design.
   
 

### Design Bug
```
SEQ_1:
begin
  if(inp_bit == 1)
    next_state = IDLE;              // <=== THE BUG
  else
    next_state = SEQ_10;
end
SEQ_10:
begin
  if(inp_bit == 1)
    next_state = SEQ_101;
  else
    next_state = IDLE;
end
SEQ_101:
begin
  if(inp_bit == 1)
    next_state = SEQ_1011;
  else
    next_state = IDLE;           // <=== THE BUG
end
```

## Design Fix
The state transition from both states SEQ_1 and SEQ_101 have been corrected. The bug-free design(relatively speaking) is in the file "seq_detect_1011_fixed.v" .


![Screenshot from 2022-08-01 11-10-04](https://user-images.githubusercontent.com/41594627/182133102-f7e03eb6-548a-4415-8685-25389ce92e23.png)
**Fig.2 A screenshot showing all test cases passed after the bugs were fixed.**





## Is The Verification Complete?
The test was carried out to verify only the functionality(behaviour) of the DUT. No verification on the timing analysis was done.
So, yes, the verification is complete for the behaviour ---in terms of state transition and output condition --- of the FSM.
