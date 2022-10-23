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
  
Of course there are better ways to wrap and unwrap our future library (so yaaaaaaayyy). For instance `every` and `some` loops would be `every` and `some` themselves. For now, the tiny library is decoupled like the following

```js
// This is our library. It takes one application to test and some configurations (exhaustion for example)

function TAOTE({ app, exhaustion }) {

    this.app = app
    this.memory = {
        // put whatever you want in general
        result: {},
        window: []
    };

    this.last = {
        args: {},
        op: undefined
    };

    var timer = function (ms) { return new Promise(resolve => { setTimeout(resolve, ms); }); };
    // Can be used to address back-pressure 
    this.rateLimit = async function* ({ generator, delay }) {
        for (let val of generator(this.app)) {
            await timer(delay);
            yield val;
        }
    };

    // Repeater returns a safe generator (that doesn't end)
    // Either it repeats itself (if finite) or repeat in round-robin
    this.repeater = function* ({ generator, round = 1 }) {
        if (!round)
            throw new Error('Round cannot be zero');
        while (true) {
            let co = 0;
            for (let val of generator(this.app)) {
                if (++co % round)
                    yield val;
                else
                    break;
            }
        }
    };
    
    // ... ... ... ... ... ... and so on ... we have some other stream operations
```

```js
// Next we need some streams !! This is an inplementation of our own, you would make your own generators with their respective specificities
function Generators() {
    // Used for recursive generators - where f(n,) is defined by f(n-1,), f(n-2,), ...
    this.base = [];
    this.fibonacci = function* fibonacci(self) {
        console.log(self.base);
        self.base.push(0);
        self.base.push(1);
        let next;
        yield 0;
        yield 1;
        while (true) {
            next = self.base[0] + self.base[1];;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
            [self.base[0], self.base[1]] = [self.base[1], next];
            yield next;
        }
    };

    this.naturals = function* naturals(self) {
        let index = 0;
        while (true)
            yield index++;
    };
    this.negatives = function* negatives(self) {
        let index = 0;
        while (true)
            yield index--;
    };

}

export default Generators
```

```js
// And now comes the MAIN we all should agree on ! You can see how TAOTE can be used (and you can suggest better ways !)
import Generators from "./Generators.js"
import TAOTE from "./TAOTE.js"


const exhaustion = 2 // Seconds we consider it Infinit already
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

const app = new Generators()
const context = new TAOTE({ app, exhaustion })
// for (let n of context.repeater({ generator: naturals, round: 5 })) {
//     console.log(n);
// }

const fib = app.fibonacci // function context detached
// for (let val of fib()) {
//     console.log(val)
// }

console.log('trying UNTIL')
for (let n of context.until({ generator: app.naturals, condition: condition1, transform, process })) {
    // console.log(n);
}
console.log('first memory')
console.log(context.memory)
// Having memory, It is easy to rebuilt a retry scenario. A scenario similar to rebooting an app after a failure.
// A retry scenario is to run the last operation with the same argumants once again but importantly building on the same previous or a new memory
for (let n of context.retry(false)) {
    // console.log(n);
}
console.log('second memory')
console.log(context.memory)
console.log('trying EVERY')
for (let _every of context.every({ generator: app.naturals, condition: condition1 })) { console.log(_every) }

console.log('trying SOME')
for (let _some of context.some({ generator: app.naturals, condition: condition2 })) { console.log(_some) }

console.log('trying SOME')
for (let _some of context.some({ generator: app.naturals, condition: condition3 })) { console.log(_some) }

let generatorA = app.naturals
let generatorB = app.negatives
console.log('trying AUntilB')
for (let n of context.AUntilB({ generatorA, generatorB, conditionA: condition1, conditionB: condition4 })) {
    // console.log(n);
}

```

## Purpose
But at the end, why is this ?   
There are a lot of studies in academia on how to test systems using temporal logic; But there is no easy and practical solutions implemented to my knowledge to be able to test everyday apps (like testing a web server, a web-app etc).   

This could help test the behaviour of apps on edge cases; Something like: what happens to an app just before overflow, what is the state of the app when unexpected values are flowing from a stream, etc. This is obviously so much ambitious, but we will go easy to have something practicale for daily usage.
