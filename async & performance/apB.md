# You Don't Know JS: Async & Performance
# Appendix B: Advanced Async Patterns

Appendix A introduced the *asynquence* library for sequence oriented async flow control, primarily based on promises and generators.

Now we'll explore other advanced asynchronous patterns built on top of our existing understanding and functionality.

## Iteratable Sequences

We introduced *asynquence*'s iterable sequences in the previous appendix, but we want to revisit them in more detail.

To refresh, recall:

```js
var domready = ASQ.iterable();

// ..

domready.val( function(){
	// DOM is ready
} );

// ..

document.addEventListener( "DOMContentLoaded", domready.next );
```

Now, let's define a sequence of multiple steps as an iterable sequence:

```js
var isq = ASQ.iterable();

isq
.then( function STEP1(x){
	return x * 2;
} )
.then( function STEP2(x){
	return x + 3;
} )
.then( function STEP3(x){
	return x * 4;
} );

isq.next( 8 ).value;	// 16
isq.next( 16 ).value;	// 19
isq.next( 19 ).value;	// 76
isq.next().done;		// true
```

As you can see, an iterable sequence is a standard-compliant *iterator* (see Chapter 4). So, it can be iterated with an ES6 `for..of` loop, just like a generator (or any other *iterable*) can:

```js
var isq = ASQ.iterable();

isq
.then( function STEP1(){ return 2; } )
.then( function STEP2(){ return 4; } )
.then( function STEP3(){ return 6; } )
.then( function STEP4(){ return 8; } )
.then( function STEP5(){ return 10; } );

for (var v of isq) {
	console.log( v );
}
// 2 4 6 8 10
```

Beyond the event triggering use-case shown in the previous appendix, iterable sequences are interesting because in essence they can be seen as a stand-in for generators or promise-chains, but with even more flexibility.

Consider a multiple Ajax request example -- we've seen the same scenario in Chapters 3 and 4, both as a promise chain and as a generator, respectively -- expressed as an iterable sequence:

```js
// sequence-aware ajax
var request = ASQ.wrap( ajax );

ASQ( "http://some.url.1" )
.runner(
	ASQ.iterable()

	.then( function STEP1(token){
		var url = token.messages[0];
		return request( url );
	} )

	.then( function STEP2(resp){
		return ASQ().gate(
			request( "http://some.url.2/?v=" + resp ),
			request( "http://some.url.3/?v=" + resp )
		);
	} )

	.then( function STEP3(r1,r2){ return r1 + r2; } )
)
.val( function(msg){
	console.log( msg );
} );
```

The iterable sequence expresses a sequential series of (potentially async) steps that looks awfully similar to a promise chain -- in other words, it's much cleaner looking than just plain nested callbacks, but not quite as nice as the `yield`-based sequential syntax of generators.

But we pass the iterable sequence into `runner(..)`, which runs it to completion the same as if it was a generator. The fact that an iterable sequence behaves essentially the same as a generator is notable for a couple of reasons.

First, iterable sequences are kind of a pre-ES6 equivalent to ES6 generators, which means you can either author them directly (to run anywhere), or you can author ES6 generators and transpile/convert them to iterable sequences (or promise chains for that matter!). Thinking of a run-to-completion generator as just syntactic sugar for a promise chain is an important recognition of their isomorphic relationship.

Before we move on, we should note that the above snippet could have been expressed in *asynquence* more simply, as:

```js
ASQ( "http://some.url.1" )
.seq( /*STEP 1*/ request )
.seq( function STEP2(resp){
	return ASQ().gate(
		request( "http://some.url.2/?v=" + resp ),
		request( "http://some.url.3/?v=" + resp )
	);
} )
.val( function STEP3(r1,r2){ return r1 + r2; } )
.val( function(msg){
	console.log( msg );
} );
```

Moreover, step 2 could have been expressed as:

```js
.gate(
	function STEP2a(done,resp) {
		request( "http://some.url.2/?v=" + resp )
		.pipe( done );
	},
	function STEP2b(done,resp) {
		request( "http://some.url.3/?v=" + resp )
		.pipe( done );
	}
)
```

So, why would we go to the trouble of expressing our flow control as an iterable sequence in a `runner(..)` step, when it seems like a simpler/flatter *asyquence* chain does the job well?

Generators, normal *asynquence* sequences, and promise chains, are all **eagerly evaluated** -- whatever flow control is expressed initially *is* the fixed flow that will be followed.

### Extending Iterable Sequences

However, iterable sequences are **lazily evaluated**, which means that during execution of the iterable sequence, you can extend the sequence with more steps if desired.

**Note:** You can only append to the end of an iterable sequence, not inject into the middle of the sequence.

Let's first look at a simpler (synchronous) example of that capability to get familiar with it:

```js
function double(x) {
	x *= 2;

	// should we keep extending?
	if (x < 500) {
		isq.then( double );
	}

	return x;
}

// setup single-step iterable sequence
var isq = ASQ.iterable().then( double );

for (var v = 10, ret;
	(ret = isq.next( v )) && !ret.done;
) {
	v = ret.value;
	console.log( v );
}
```

The iterable sequence starts out with only one defined step (`isq.then(double)`), but the sequence keeps extending itself under certain conditions (`x < 500`). Both *asynquence* sequences and promise chains technically *can* do something similar, but we'll see in a little bit why their capability is insufficient.

Though this example is rather trivial and could otherwise be expressed with a `while` loop in a generator, we'll consider more sophisticated cases.

For instance, you could examine the response from an Ajax request and if it indicates that more data is needed, you conditionally insert more steps into the iterable sequence to make the additional request(s). Or you could conditionally add a value-formatting step to the end of your Ajax handling.

Consider:

```js
var steps = ASQ.iterable()

.then( function STEP1(token){
	var url = token.messages[0].url;

	// was an additional formatting step provided?
	if (token.messages[0].format) {
		steps.then( token.messages[0].format );
	}

	return request( url );
} )

.then( function STEP2(resp){
	// add another Ajax request to the sequence?
	if (/x1/.test( resp )) {
		steps.then( function STEP5(text){
			return request(
				"http://some.url.4/?v=" + text
			);
		} );
	}

	return ASQ().gate(
		request( "http://some.url.2/?v=" + resp ),
		request( "http://some.url.3/?v=" + resp )
	);
} )

.then( function STEP3(r1,r2){ return r1 + r2; } );


ASQ( {
	url: "http://some.url.1",
	format: function STEP4(text){
		return text.toUpperCase();
	}
} )
.runner( steps )
.val( function(msg){
	console.log( msg );
} );
```

Can the flexibility (conditional behavior) of the `steps` iterable sequence be expressed with a generator? Kind of, but we have to rearrange the logic in a slightly awkward way:

```js
function *steps(token) {
	// **STEP 1**
	var resp = yield request( token.messages[0].url );

	// **STEP 2**
	var rvals = yield ASQ().gate(
		request( "http://some.url.2/?v=" + resp ),
		request( "http://some.url.3/?v=" + resp )
	);

	// **STEP 3**
	var text = rvals[0] + rvals[1];

	// **STEP 4**
	// was an additional formatting step provided?
	if (token.messages[0].format) {
		text = yield token.messages[0].format( text );
	}

	// **STEP 5**
	// need another Ajax request added to the sequence?
	if (/foobar/.test( resp )) {
		text = yield request(
			"http://some.url.4/?v=" + text
		);
	}

	return text;
}

// note: `*steps()` can be run by the same `ASQ` sequence
// as `steps` was previously
```

Setting aside the already identified benefits of the sequential, synchronous-looking syntax of generators (see Chapter 4), the `steps` logic had to be reordered in `*steps()`, to fake the dynamicism of the extendable iterable sequence `steps`.

What about promises or sequences, though? You *can* do something like this:

```js
var steps = something( .. )
.then( .. )
.then( function(..){
	// ..

	// extending the chain, right?
	steps = steps.then( .. );

	// ..
})
.then( .. );
```

The problem is subtle but important to grasp. So, consider trying to wire up our `steps` promise-chain into our main program flow (this time expressed with promises instead of *asynquence*):

```js
var main = Promise.resolve( {
	url: "http://some.url.1",
	format: function STEP4(text){
		return text.toUpperCase();
	}
} )
.then( function(..){
	return steps;			// hint!
} )
.val( function(msg){
	console.log( msg );
} );
```

Can you spot the problem now? Look closely!

There's a race condition for sequence steps ordering. When you `return steps`, at that moment `steps` *might* be the originally defined promise chain, or it might now point to the extended promise chain via `steps = steps.then(..)`, depending on what order things happen.

Here are the two possible outcomes:

1. If `steps` is still the original promise chain, once it's "extended" by `steps = steps.then(..)`, that extended promise on the end of the chain is **not** considered by `main`, as it's already tapped the `steps` chain.
2. If `steps` is already the extended promise chain, it works as we expect in that the extension promise is what `main` taps to wire in the `steps` chain.

Other than the obvious fact that a race condition is intolerable, case #1 is the concern; it illustrates **eager evaluation** of the promise chain. We easily extended the iterable sequence without issue, because iterable sequences are **lazy evaluated**.

The more dynamic you need your flow control, the more iterable sequences will shine.

## Summary

Promises and generators provide the foundational building blocks upon which we can build much more sophisticated and capable asynchrony.
