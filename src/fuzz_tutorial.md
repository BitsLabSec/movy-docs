# A Tour of Invariant Testing

This chapter will cover the invariant testing step by step.

Please ensure you have installed Movy following our [instructions](./install.md).

The code sample could be found at [here](https://github.com/BitsLabSec/movy/tree/master/test-data/counter). We will use the Sui flavor in this tutorial.

## Code Under Testing

Assume a small code snippet that maintains a counter:

```move
module counter::counter;

public struct Counter has key {
    id: UID,
    value: u64
}

public fun create(ctx: &mut TxContext) {
    let counter = Counter {
        id: object::new(ctx),
        value: 1
    };
    transfer::share_object(counter);
}

public fun increment(counter: &mut Counter, n: u64) {
    counter.value = counter.value + n;
}

public fun value(counter: &Counter): u64 {
    counter.value
}
```

In general, every object shared by this modules contains a `u64` value. It is obvious that **the value is always positive** and now we would like to verify this via Movy.

## Add Movy

First, we add the following lines to the `Move.toml`.

```toml
[dev-dependencies]
movy = {git = "https://github.com/BitsLabSec/movy", subdir = "move/movy", rev = "master"}
```

This allows your code to interact with Movy.

> Note the Movy dependency is added to `dev-dependencies`, i.e. the Movy code will never really go live and does not affect the integrity of the existing modules.

## Deploy Your Package

The very first step is deploying the target packages. In the `counter` example above, Movy need at least one `Counter` object so that we could test the functions. To create shared objects (and possibly do other setup), add these lines in `tests/movy.move` with the target package.

> Note the Movy modules are organized as unit tests and will only exist in the local emulated environment. It is worth mentioning that Sui rejects any testing modules or testing functions.

```move
#[test_only]
module counter::counter_tests;

use sui::test_scenario::{Self as ts};
use counter::counter::{Self, Counter};
use sui::bag::Self;

#[test]
public fun movy_init(
    deployer: address,
    attacker: address
) {
    let mut scenario = ts::begin(deployer);
    {
        ts::next_tx(&mut scenario, deployer);
        counter::create(ts::ctx(&mut scenario));
    };

    ts::next_tx(&mut scenario, attacker);
    {
        let mut counter_val = ts::take_shared<Counter>(&scenario);
        counter::increment(&mut counter_val, 0);
        ts::return_shared(counter_val);
    };

    ts::end(scenario);
}
```

We will break down the code snippet:

- `#[test_only]` ensures that the module itself will never be used in a production environment and enables [test_scenario](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/packages/sui-framework/sources/test/test_scenario.move), which further allows us to do **multi transaction** deployment.
- `movy_init` is a special function in Movy that will be called after the module is deployed. The `deployer` will be the one that deployed **the target module**, in this case, the `counter` module while the `attacker` will be the one trying to attack the module. In general, a `movy_init` should always transfer administration capabilities to `deployer` while grants minimal permission to `attacker` for a real-world scenario.
- `ts::begin`, `ts::next_tx` and `ts::end` starts a multi-transaction testing scenario.
- `ts::take_shared` and `ts::return_shared` create a shared object and use it immediately with `increment`, which will _only_ works in a testing module with `testing_scenario`. This essentially simulates two transactions of the deployment because shared objects are not possible to create and use within a single transaction.

Movy will call `movy_init` during fuzzing and `movy_init` is supposed to setup the target modules. All changes will be captures and the shared objects created will be used in the following fuzzing campaign.

If nothing special is needed, `movy_init` could be omitted or empty.

## Write First Movy Test

A Movy test has some special setup compared to ordinary Move tests. The following code presents a minimal Movy test in file `tests/movy.move`.

```Move
use movy::oracle::crash_because;

#[test]
fun extract_counter(ctr: &Counter): (ID, u64) {
    let val = counter::value(ctr);
    let ctr_id = sui::object::id(ctr);
    (ctr_id, val)
}

#[test]
public fun movy_pre_increment(
    movy: &mut context::MovyContext,
    ctr: &mut Counter,
    _n: u64
) {
    let (ctr_id, val) = extract_counter(ctr);
    let state = context::borrow_mut_state(movy);
    bag::add(state, ctr_id, val);
}

#[test]
public fun movy_post_increment(
    movy: &mut context::MovyContext,
    ctr: &mut Counter,
    n: u64
) {
    let (ctr_id, new_val) = extract_counter(ctr);
    let state = context::borrow_state(movy);
    let previous_val = bag::borrow<ID, u64>(state, ctr_id);
    if (*previous_val + n != new_val) {
        crash_because(b"Increment does not correctly inreases internal value.".to_string());
    }
}
```

Again, we can break down the code:

- `movy_pre_increment` and `movy_post_increment` are special functions that tell Movy to call them before and after `increment` function. In other words, if Movy initiates a call to `counter::increment`, it will inserts the two functions calls to make the transaction sequence like `[movy_pre_increment, counter::increment, movy_post_increment]`. Movy identifies such functions with a pattern like `movy_pre_*` and `movy_post_*`.
- `context::MovyContext` is a fresh wrapper of `sui::bag::Bag` that allows users to store any context. In this case, we store the value before increment. It should be always the first argument of the `movy_pre_` and `movy_post_` functions.
- `ctr: &mut Counter, n: u64` is the original signature of the `counter::increment`.
- `movy::oracle::crash_because` will inform Movy that an invariant violation happens and Movy will capture the seed as a crash.
- The core logic of the invariant is that, before the `counter::increment` call, we store the value in `movy_pre_increment`. After the `counter::increment` call, we get the value and check if the value exactly increases by `n`. If not, `movy::oracle::crash_because` will tell Movy to save this seed.

## Run Movy

Finally, it is time to test the invariants we just wrote:

```bash
RUST_LOG=info ./target/release/movy sui fuzz
    -l ./test-data/counter
    -o /tmp/out
```

- `RUST_LOG=info` will print some helpful logs.
- `-l ./test-data/counter` points to the target package with the Movy test modules.
- `-o /tmp/out` will tell Movy to save outputs to `/tmp/out` and we will cover this later.

You could see logs like:

```
[INFO  movy_replay::env] Committing testing std 0x000000000000000000000000000000000000000000000000000000000000000b
[INFO  movy_replay::env] Committing testing std 0x0000000000000000000000000000000000000000000000000000000000000001
[INFO  movy_replay::env] Committing testing std 0x0000000000000000000000000000000000000000000000000000000000000002
[INFO  movy_replay::env] Committing testing std 0x0000000000000000000000000000000000000000000000000000000000000003
...
[INFO  movy::sui::env] Deploying the local package at ./test-data/counter/
[INFO  movy_replay::env] Compiling ./test-data/counter/ with test mode...
[INFO  movy_replay::env] Detected a movy_init at: 0x9ae10865d456c2a9ebc47b754db3f77b96eebb049192a26fce0577aaca3a5e2a::counter_tests
[INFO  movy_replay::env] Commiting movy_init effects...
...
[INFO  movy_fuzz::operations::sui_fuzz] [Client Heartbeat #0] run time: 33s, clients: 1, corpus: 3, objectives: 0, executions: 326, exec/sec: 9.632, code-fb: 68/16384 (0%), stability: 1/1 (100%)
```

Generally, Movy does the following things to test the contracts.

- First, Movy will spin up an empty fork of the Chain, in this case, the Sui chain.
- Movy will deploy the Sui standard framework to the fork in the testing mode, which enables the `0x1::unit_test` and `0x2::test_scenario`.
- Then Movy will invoke the Move compiler to compile the given project `./test-data/counter` to obtain modules and their interfaces.
- Movy further will deploy the modules to our fork (at `0x9ae10865d456c2a9ebc47b754db3f77b96eebb049192a26fce0577aaca3a5e2a`) and execute `movy_init` in the testing modules.
- Once everything is ready, Movy starts to assemble random transactions. Gnerally, the `corpus` metrics indicates the number of interesting inputs, the `objectives` indicates the number of crashing inputs that violate invariants and `code-fb` refers to the code coverage.

## Trigger a Violation

The fuzzing in the previous section shall work, but not really trigger a crash because the invariant always holds. Now let's manually add a bug for our counter implementation:

```
public fun increment(counter: &mut Counter, n: u64) {
-    counter.value = counter.value + n;
+    counter.value = counter.value + 1;
}
```

Note we intentionally broke the invariant: only increase `1` though users request to increas `n`. Rerun Movy with the project and we could see:

```
...
[INFO  movy_fuzz::operations::sui_fuzz] [Objective #0] run time: 24s, clients: 1, corpus: 2, objectives: 3, executions: 247, exec/sec: 10.27, code-fb: 104/16384 (0%), stability: 1/1 (100%), crash-fb: 118/16384 (0%)
```

> In case you see errors like "The given output is already there....", rerun Movy with `-f` parameter to automatically remove the existing results. The mechanism is to prevent accidental removal of previous fuzzing campaigns.

Movy immediately could find a `objectives` and this indicates that we found the violations.

## Replay and Inspect a Violation

In addition to finding violations, Movy also supports inspect how violation happens by _replaying the violations_. Recall that we have the `-o /tmp/out` option to Movy to save outputs to `/tmp/out` and it is time to see the contents.

```bash
> ls /tmp/out
args.json  crashes/  env.bin  fuzz_meta.json  queue/
```

- `args.json` is the arguments that start Movy, i.e., the CLI parameters.
- `fuzz_meta.json` is the metadata we setup for the fuzzing campaign, including the target contracts and their interfaces.
- `env.bin` is a binary file that holds our forked chain contents.
- `queue/` saves the interesting seeds and we can ignore it in this tutorial.
- `crashes/` saves the violations Movy have found.

Since we have found some violations, we shall have at least `crashes/0.json` and we can replay it by providing the saved environment:

```
RUST_LOG=info ./target/release/movy sui replay-seed \
        -s /tmp/out/crashes/0.json \
        -e /tmp/out/env.bin \
        -m /tmp/out/fuzz_meta.json \
        --trace
```

> The output of a full trace is usually very long. Saving to a text file and using an editor is highly recommended.

This will print a full trace including everything during execution and we could see that:

```
├─ 0x977654ad5e98ce5a09b7bbac3421b312aef7370b0bfc889e6181a8bcc23d8b9c:counter_tests:movy_post_increment(...)
...
│  └─ 0x977654ad5e98ce5a09b7bbac3421b312aef7370b0bfc889e6181a8bcc23d8b9c:oracle:crash_because(...)
```

So the invariant violation happens exactly in `movy_post_increment`.