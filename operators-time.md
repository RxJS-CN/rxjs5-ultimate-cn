# Operators time
There are plenty of operators dealing with time in some way such as `delay` `debounce` `throttle` `interval` etc.

This is not an easy topic. There are many areas of application here, either you might want to synchronize responses from APIS or you might want to deal with other types of streams such as events like clicks or keyup in a UI.
## delay
`delay()` is an operator that delays every value being emitted
Quite simply it works like this :
```
var start = new Date();
let stream$ = Rx.Observable.interval(500).take(3);

stream$
.delay(300)
.subscribe((x) => {
    console.log('val',x);
    console.log( new Date() - start );
})

//0 800ms, 1 1300ms,2 1800ms
```


### Business case
Delay can be used in a multitude of places but one such good case is when handling errors especially if we are dealing with `shaky connections` and want it to retry the whole stream after x miliseconds:

```
let values$ = Rx.Observable.interval(1000).take(5);
let errorFixed = false;

values$
.map((val) => {
    if(errorFixed) { return val; }
    else if( val > 0 && val % 2 === 0) {
        errorFixed = true;
        throw { error : 'error' };

    } else {
        return val;
    }
})
.retryWhen((err) => {
    console.log('retrying the entire sequence');
    return err.delay(200);
})
.subscribe((val) => { console.log('value',val) })

// 0 1 'wait 200ms' retrying the whole sequence 0 1 2 3 4
```
The `delay()` operator is used within the `retyWhen()` to ensure that the retry happens a while later to in this case give the network a chance to recover.



## sample
I usually think of this scenario as *talk to the hand*.
What I mean by that is that events are only fired at specific points 

### Business case
So the ability to ignore events for x miliseconds is pretty useful. Imagine a save button being repeatedly pushed. Wouldn't it be nice to only act after x miliseconds and ignore the other pushes ?

```
const btn = document.getElementById('btnIgnore');
var start = new Date();

const input$ = Rx.Observable
  .fromEvent(btn, 'click')

  .sampleTime(2000);

input$.subscribe(val => {
  console.log(val, new Date() - start);
});
```
The code above does just that.


## debounceTime
TODO
### Business case
TODO

## throttleTime
TODO

## buffer
This operator has the ability to record x number of emitted values before it outputs its values, this one comes with one or two input parameters.

```
.buffer( whenToReleaseValuesStartObservable )

or

.buffer( whenToReleaseValuesStartObservable, whenToReleaseValuesEndObservable )

```

So what does this mean?
It means given we have for example a click of streams we can cut it into nice little pieces where every piece is equally long. Using the first version with one parameter we can give it a time argument, let's say 500 ms. So something emits values for 500ms then the values are emitted and another Observable is started, and the old one is abandoned. It's much like using a stopwatch and record for 500 ms at a time. Example :

```
let scissor$ = Rx.Observable.interval(500)

let emitter$ = Rx.Observable.interval(100).take(10) // output 10 values in total
.buffer( scissor$ )

// [0,1,2,3,4] 500ms [5,6,7,8,9]
```

Marble diagram

```
--- c --- c - c --- >
-------| ------- |- >
Resulting stream is :
------ r ------- r r  -- > 
```

### Business case
So whats the business case for this one?
`double click`, it's obviously easy to react on a `single click` but what if you only want to perform an action on a `double click` or `triple click`, how would you write code to handle that? You would probably start with something looking like this :

```

$('#btn').bind('click', function(){
  if(!start) { start = timer.start(); }
  timePassedSinceLastClickInMs = now - start;
  if(timePassedSinceLastClickInMs < 250) {
     console.log('double click');
       
  } else {
     console.log('single click')
  }
  
  start = timer.start();  
})
```
Look at the above as an attempt at pseudo code. The point is that you need to keep track of a bunch of variables of how much time has passed between clicks. This is not nice looking code, lacks elegance

#### Model in Rxjs
By now we know Rxjs is all about streams and modeling values over time.
Clicks are no different, they happen over time.
```
---- c ---- c ----- c ----- >
```
We however care about the clicks when they appear close together in time, i.e as double or triple clicks like so :

 ```
 --- c - c ------ c -- c -- c ----- c 
 ``` 
 From the above stream you should be able to deduce that a `double click`, one `triple click` and one `single click` happened.

So let's say I do, then what?
You want the stream to group itself nicely so it tells us about this, i.e it needs to emit these clicks as a group. `filter()` as an operator lets us do just that. If we define let's say 300ms is a long enough time to collect events on, we can slice up our time from 0 to forever in chunks of 300 ms with the following code:  

```
let clicks$ = Rx.Observable.fromEvent(document.getElementById('btn'), 'click');

let scissor$ = Rx.Observable.interval(300);

clicks$.buffer( scissor$ )
      //.filter( (clicks) => clicks.length >=2 )
      .subscribe((value) => {
          if(value.length === 1) {
            console.log('single click')
          }
          else if(value.length === 2) {
            console.log('double click')
          }
          else if(value.length === 3) {
            console.log('triple click')
          }
          
      });
```

Read the code in the following way, the buffer stream, `clicks$` will emit its values every 300ms, 300 ms is decided by `scissor$` stream. So the `scissor$` stream is the scissor, if you will, that cuts up our click stream and voila we have an elegant `double click` approach. As you can see the above code captures all types of clicks but by uncommenting the `filter()` operation we get only `double clicks` and `triple clicks`. 

`filter()` operator can be used for other purposes as well like recording what happened in a UI over time and replay it for the user, only your imagination limits what it can be used for.

