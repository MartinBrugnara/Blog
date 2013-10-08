Performant synchronized scrolling with heavy HTML components
============================================================

The Problem
------------
I had to graphically represent the situation of an accomodation.  

For each day of the year I have to display the state of a few hundreds
rooms, pitches, parkings and other various resource.  
So what better than a table? Nothing.... but....

Over that I should represent many kinde of things such as overnight stays, rents,
bookings.  
And the operator should be able to manage them.  

So I decided to move to div.  
To allow operators to manage this items and interact with them I used jQuery UI
 drag&drop and resize support.  
That choice cost me the overhead of two more children for each child.  
So now in the worst case I would had 54000 div 
each one with it's style and it's event listeners attached.  

It was slow and there was an heavy latency while trying to operate.
The view was constantly 'destroyed':  
rendering error, partial rendering and so on.  

As you can imagine that was not acceptable for me, and I hope for you too.  

So I started to find a way to hack this feature.  

> Maybe you yet knew what I'am going to introduce anyway  
> I would like to share my new knowledge.  
> So here is what I got out.  



Suppose to have an headerPane and an asidePane.  
Example:
- Full year calendar [Days, Hours].  
- Big zoomed image with two helper that indicate the current position.  
- Three overlapping div with thousand of children.  (My case)  

Common implementation
---------------------
```html
...
<div id="scrollpane">
    <div id="headerPane"></div>
    <div id="asidePane"></div>
    <div id="dataPane"></div>
</div>
...
```

```css
#dataPane {
    owerflow: scroll;
}

#headerPane, #asidePane {
    owerflow: hidden;
}
```

```javascript
$('#dataPane').on('scroll', function(e) {
    $('#headerPane').scrollLeft(e.currenTarget.scrollLeft);
    $('#asidePane').scrollTop(e.currenTarget.scrollTop);
})
```

### Main problems with this implementation
1. Slow scrolling.
2. Update delay.
3. Paint delay.
4. View destroyed. (Like when you work without vSync)

Solution so far
---------------
```html
...
<div id="scrollpane">
    <div id="headerPane"></div>
    <div id="asidePane"></div>
    <div id="dataPane"></div>
    <div id="scroller"></div> <!-- +++ --->
</div>
...
```

```css
#dataPane, #headerPane, #asidePane {
    owerflow: hidden;
    display: block;
}
#scroller {
    owerflow: scroll;
    display: block;
}
```

```javascript

// Config
    var w = 10; // horizontal tick (1 == direct/no tick)
    var h = 10; // vertical tick (1 == direct/no tick)

// Components bindig
    var hp = $('#headerPane');
    var vp = $('#asidePane');
    var dp = $('#dataPane');
    var sc = $('#scroller');
    var ct = sc.get(0);

--------------------------------------------------------------------------------
// Binding scroller & init
    dp.on('resize', function(){
        sc.width(dp.width());
        sc.height(dp.height());
    }).resize();

// Let mouse scrolling work
    var scroller = $('#scroller');
    dp.on('mousewheel', function(e) {
      var delta = e.originalEvent.wheelDelta/120;
      sc.scrollTop(sc.scrollTop() - delta*w); 
      // TIP: (+ delta) to simulate natural scrolling
    });

// Algorithm
    var ls_left=0, ls_top=0;
    sc.on('scroll', function() {
      // horizontal scroll
      var sl = ct.scrollLeft;
      var c_left;
      if (sl == ct.scrollWidth) {
        // left border
        c_left = ct.scrollWidth;
      } else {
        // next tick
        c_left = Math.round(sl/w)*w;
      }
      if (c_left !== ls_left) {
        hp.scrollLeft(c_left);
        dp.scrollLeft(c_left);
        ls_left = c_left;
      } 

      // vertical scroll
      var st = ct.scrollTop;
      var c_top;
      if (st == ct.scrollHeight) {
        c_top = ct.scrollHeight;
      } else {
        c_top = Math.round(st/h)*h;
      }
      if (c_top !== ls_top) {
        vp.scrollTop(c_top);
        dp.scrollTop(c_top);
        ls_top = c_top;
      }
    });
```

### Summary
- [X] Slow scrolling.  
    Until your div is empty (like our #scroller) it's very light to move around.
- [X] Update delay.  
    Now we set the scroll values simultaneously (more or less).
- [X] Paint delay.  
    For as we set now the scroll values the repaint will be fired on both areas almost simultaneously.  
- [X] View destroyed.  
    Having this kind of synchronization, seems to be enough to maintain the view's integrity


What TODO next
--------------
If this is not enough maybe you would like to implement a system based on regions/tiles.

Point of view: while you are not using it, do not care about it.  

> Keep only the components of the (x-1), x and (x+1) regions in the #dataPane.  
> Add the components of the next region and discard the components of the more distant one;  
> Leaving the browser to render only one part of your data, letting it working faster.  

P.s.
====
* If you know a better way to do it, please let know =)

:q
