# Test Apps On The Edge

This is an attempt to test apps the time they bootstrap, crash or recover. To be able to deliver such practical solution, let's rephrase the previous phrase to the following:

Take decisions on un-bound data under the [closed world assumption](https://en.wikipedia.org/wiki/Closed-world_assumption) 

## Introduction
Streams are generated data in real-time, which means we do not have full controle on data (absolutely). We can have control over under "asymptions" though.

Let me illustrate quickly. Arrays are bound, we can apply `Array.filter`, `Array.map`, `Array.every`, `Array.some` etc on. For streams, it is more suitable to apply the concept of [closed world assumption](https://en.wikipedia.org/wiki/Closed-world_assumption). So for our case:  

`Array.every([1,2,3,4,5,6,7,8,┴], (e) => e < 10)` would be true. Note that `┴` means a contradiction. Just another way not to say the boolean `false` but something out of control like time or ressources are exhausted. This is in fact how we would implement such system; We need to emulate such even to be able to implement our idea.

## Example

Note in our example:
  - I'm using generators as a simple form of streams
  - `getNaturals` as a stream of natural numbers
  - condition is even numbers
  - Everything else is an example
  
Of course there are better ways to wrap and unwrap our future library (so yaaaaaaayyy). For instance `every` and `some` loops would be `every` and `some` themselves. 

```js
function BooleanStream() {

    this.memory = {
        // put whatever you want in general
        result: {},
        window: []
    }

    this.last = {
        args: {},
        op: undefined
    }

    var timer = function (ms) { return new Promise(resolve => { setTimeout(resolve, ms) }) }
    // Can be used to address back-pressure 
    this.rateLimit = async function* ({ generator, delay }) {
        for (let val of generator()) {
            await timer(delay)
            yield val
        }
    }

    // Repeater returns a safe generator (that doesn't end)
    // Either it repeats itself (if finite) or repeat in round-robin
    this.repeater = function* ({ generator, round = 1 }) {
        if (!round) throw new Error('Round cannot be zero')
        while (true) {
            let co = 0
            for (let val of generator()) {
                if (++co % round) yield val
                else break
            }
        }
    }

    // So generators when defined are pure, let's add randomness possibility for tests 
    // This does not change values, we are not talking about that.
    // This changes the structure (when it ends)
    // Only one of the defined types must be provided
    // Note that original positions are preserved, and the new impure generator has not only data, but index as well
    // TODO:
    this.impurify = function* ({ generator, endBeforeStart = false, ignoreSome = 0, hop = 0, expander = 0, shrinker = 0 }) {
        if (!(endBeforeStart || ignoreSome || hop || expander || shrinker)) throw new Error('One method must be provided')
        if (endBeforeStart && (ignoreSome || hop || expander || shrinker)) throw new Error('endBeforeStart cannot be combined with other methods')
        if (shrinker && expander) console.warn('shrinker along with expander is confusing')

        if (endBeforeStart) yield null
        if (ignoreSome) {
            let position = 0
            for (let val of generator()) {
                if (-(ignoreSome--) || ignoreSome == 0)
                    yield [position, val]
                position++
            }
            return
        }

        if (hop) {
            let co = 0
            let position = 0
            for (let val of generator()) {
                if (co++ % hop)
                    yield [position, val]
                position++
            }
            return
        }

        if (expander) {
            let co = 0
            let position = 0
            for (let val of generator()) {
                if (co++ % 2) {
                    for (let i = 0; i < expander; i++) {
                        yield [-1, Math.random()]
                    }
                }
                yield [position, val]
                position++
            }
        }
        // TODO:
        if (shrinker) { 0 }

        throw new Error('This should not happen. Verify parameters types')
    }

    this.retry = function (resetMemory) {
        if (resetMemory) this.memory = { result: {}, window: [] }
        return this[this.last.op](this.last.args)
    }

    // Returns results as long as condition IS MET 
    this.until = function* ({ generator, condition = (_) => true, transformer = (v) => v, process = (v, m) => null, window = 1 }) {
        this.last.args = { generator, condition, transformer, process, window }
        this.last.op = 'until'
        let startTime = Date.now()
        let result
        let co = 0
        for (let val of generator()) {
            if (result)
                return result
            if ((Date.now() > (startTime + exhaustion * 1000))) {
                result = 'exhausted'
                yield result
            }
            if (condition(val)) {
                this.memory.window[co++ % window] = val
                process(val, this.memory)
                yield transformer(val)
            } else {
                result = 'halted'
                yield result
            }
        }
        console.log('done')
    }

    // Checks if every value fulfills a condition and returns a Boolean
    // Returns as soon as condition is False
    // Returns True when exhausted
    this.every = function* ({ generator, condition = (_) => true }) {
        this.last.args = { generator, condition }
        this.last.op = 'every'
        let startTime = Date.now()
        let result
        for (let val of generator()) {
            if (result) {
                yield result
                return
            }
            if ((Date.now() > (startTime + exhaustion * 1000))) {
                result = 'exhausted'
                yield true
            }
            // Do not return from here as it is the default case of Every
            if (condition(val)) {
                1
            } else {
                result = 'halted'
                yield false
            }
        }
        console.log('done')
    }

    // Checks if some values fulfill a condition and returns a Boolean
    // Returns as soon as condition is True
    // Returns False when exhausted
    this.some = function* ({ generator, condition = (_) => false }) {
        this.last.args = { generator, condition }
        this.last.op = 'some'
        let startTime = Date.now()
        let result
        for (let val of generator()) {
            if (result) {
                yield result
                return
            }
            if ((Date.now() > (startTime + exhaustion * 1000))) {
                result = 'exhausted'
                yield false
            }
            // Do not return from here as it is the default case of Some
            if (!condition(val)) {
                1
            } else {
                result = 'halted'
                yield true
            }
        }
        console.log('done')
    }

    // Composition (inspired from LTL: Linear Time Logic. A logic used for formal testing systems)
    this.AUntilB = function* ({ generatorA, generatorB, conditionA = (_) => false, conditionB = (_) => false }) {
        this.last.args = { generatorA, generatorB, conditionA, conditionB }
        this.last.op = 'AUntilB'
        let startTime = Date.now()
        let result

        const combined = function* (genA, genB) {
            let nextGenA, nextGenB
            while (!(nextGenA = genA.next()).done && !(nextGenB = genB.next()).done) {
                yield { a: nextGenA.value, b: nextGenB.value }
            }
        }(generatorA(), generatorB())

        for (let val of combined) {
            if (result)
                return result
            if ((Date.now() > (startTime + exhaustion * 1000))) {
                result = 'exhausted'
                yield result
            }
            if (conditionB(val.b)) {
                yield val.b
            } else if (conditionA(val.a)) {
                yield val.a
            }
        }
        console.log('done')
    }

}

const exhaustion = 2 // Seconds we consider it Infinit already

function* naturals() {
    for (let index = 0; index < Infinity; index++) {
        yield index
    }
}
function* negatives() {
    for (let index = 0; index < Infinity; index++) {
        yield -index
    }
}

const condition1 = (value) => value == 0 || value > 0
const condition2 = (value) => value > 300
const condition3 = (value) => value < 0
const condition4 = (value) => value < -300

function transform(value) {
    return value * 2
}

function process(value, memo) {
    // Do whatever and rely on memo.result to stack results
    memo.result['sum'] = memo.result['sum'] ? (memo.result['sum'] + value) : 0
}


const booleanStream = new BooleanStream()
// for (let n of booleanStream.repeater({ generator: naturals, round: 5 })) {
//     console.log(n);
// }

console.log('trying UNTIL')
for (let n of booleanStream.until({ generator: naturals, condition: condition1, transform, process })) {
    // console.log(n);
}
console.log('first memory')
console.log(booleanStream.memory)
// Having memory, It is easy to rebuilt a retry scenario. A scenario similar to rebooting an app after a failure.
// A retry scenario is to run the last operation with the same argumants once again but importantly building on the same previous or a new memory
for (let n of booleanStream.retry(false)) {
    // console.log(n);
}
console.log('second memory')
console.log(booleanStream.memory)
console.log('trying EVERY')
for (let _every of booleanStream.every({ generator: naturals, condition: condition1 })) { console.log(_every) }

console.log('trying SOME')
for (let _some of booleanStream.some({ generator: naturals, condition: condition2 })) { console.log(_some) }

console.log('trying SOME')
for (let _some of booleanStream.some({ generator: naturals, condition: condition3 })) { console.log(_some) }

let generatorA = naturals
let generatorB = negatives
console.log('trying AUntilB')
for (let n of booleanStream.AUntilB({ generatorA, generatorB, conditionA: condition1, conditionB: condition4 })) {
    // console.log(n);
}
```

## Purpose
But at the end, why is this ?   
There are a lot of studies in academia on how to test systems using temporal logic; But there is no easy and practical solutions implemented to my knowledge to be able to test everyday apps (like testing a web server, a web-app etc).   

This could help test the behaviour of apps on edge cases; Something like: what happens to an app just before overflow, what is the state of the app when unexpected values are flowing from a stream, etc. This is obviously so much ambitious, but we will go easy to have something practicale for daily usage.