---
layout: post
title: 阅读摘要之 You Don't Need jQuery!
---

代码全部摘录自下面的博客文章:

- [Selecting Elements](http://blog.garstasio.com/you-dont-need-jquery/selectors/)
- [DOM Manipulation](http://blog.garstasio.com/you-dont-need-jquery/dom-manipulation/)
- [Ajax Requests](http://blog.garstasio.com/you-dont-need-jquery/ajax/)
- [Events](http://blog.garstasio.com/you-dont-need-jquery/events/)
- [Utilities](http://blog.garstasio.com/you-dont-need-jquery/utils/)


## Selecting Elements

### By ID

~~~javascript
// jQuery
$('#myElement');

// DOM API IE 5.5+
document.getElementById('myElement');
// DOM API IE 8+
document.querySelector('#myElement');
~~~

### By Attribute

~~~javascript
// jQuery
$('[data-foo-bar="someval"]');

// DOM API IE 8+
document.querySelectorAll('[data-foo-bar="someval"]');
~~~

### By Pseudo-class

~~~javascript
// jQuery
$('#myForm :invalid');

// IE 8+
document.querySelectorAll('#myForm :invalid');
~~~

### Children

~~~javascript
// jQuery
$('#myParent').children();
// jQuery find children with an attribute of "ng-click"
$('#myParent').children('[ng-click]');
$('#myParent > [ng-click]');

// DOM API IE 5.5+
// NOTE: This will include comment and text nodes as well.
document.getElementById('myParent').childNodes;

// DOM API IE 9+ (ignores comment & text nodes).
document.getElementById('myParent').children;
// DOM API IE 8+
document.querySelector('#myParent > [ng-click]');
~~~

### Descendants

~~~javascript
// jQuery
$('#myParent A');

// DOM API IE 8+
document.querySelectorAll('#myParent A');
~~~

### Excluding Elements

~~~javascript
// jQuery
$('DIV').not('.ignore');
// or
$('DIV:not.(.ingnore)');

// DOM API IE 9+
document.querySelectorAll('DIV:not(.ignore)');
~~~

### Multiple Selectors

~~~javascript
// jQuery
$('DIV, A, SCRIPT');

// DOM API IE 8+
document.querySelectorAll('DIV, A, SCRIPT');
~~~

### 自己做一个 $

~~~javascript
window.$ = function(selector) {
  var selectorType = 'querySelectorAll';

  if (selector.indexOf('#') === 0){
    selectorType = 'getElementById';
	selector = selector.substr(1, selector.length);
  }

  return document[selectorType](selector);
}
~~~
## DOM Manipulation

### Creating Elements

~~~javascript
// jQuery
$('<div></div>');

// DOM API IE 5.5+
document.createElement('div');
~~~

### Inserting Elements Before & After

~~~javascript
// jQuery
$('#1').after('<div id="1.1"></div>');

// DOM API IE 4+
document.getElementById('1)
  .insertAdjacentHTML('afterend', '<div id="1.1"></div>');
  
// jQuery
$('#1').before('<div id="0.9"></div>');

// DOM API IE 4+
document.getElementById('1')
  .insertAdjacentHTML('beforebegin', '<div id="0.9"></div>');
~~~

### Inserting Elements As Children

~~~javascript
// jQuery
$('#parent').prepend('<div id="newChild"></div>');

// DOM API IE 4+
document.getElementById('parent')
  .insertAdjacentHTML('afterbegin', '<div id="newChild"></div>');
  
// jQuery
$('#parent').append('<div id="newChild"></div>');
// DOM API IE 4+
document.getElementById('parent')
  .insertAdjacentHTML('beforeend', '<div id="newChild"></div>');
~~~

### Moving Elements

~~~javascript
// jQuery
$('#parent').append('#orphan');

// DOM API IE 5.5+
document.getElementById('parent)
  .appendChild(document.getElementById('orphan'));
  
// jQuery
$('#parent').prepend($('#orphan'));

// DOM API IE 5.5+
document.getElementById('parent')
  .insertBefore(document.getElementById('orphan'), document.getElementById('c1'));
~~~

### Removing Elements

~~~javascript
// jQuery
$('#foobar').remove();

// DOM API IE 5.5+
document.getElementById('foobar').parentNode
  .removeChild(document.getElementById('foobar'));
~~~

### Adding & Removing CSS Classes

~~~javascript
// jQuery
$('#foo').addClass('bold');

// DOM API, ALL modern browsers, with the exception of IE9
document.getElementById('foo').classList.add('bold');

// All browsers
document.getElementById('foo').className += 'bold';

// jQuery
$('#foo').removeClass('bold');

// DOM API, All modern browsers, with the exception of IE9
document.getElementById('foo').classList.remove('bold');
// DOM API, All browsers
document.getElementById('foo').className =
  document.getElementById('foo').className.replace(/^bold$/, '');
~~~

### Adding/Removing/Changing Attributes

~~~javascript
// jQuery
$('#foo').attr('role', 'button');

// DOM API IE 5.5+
document.getElementById('foo').setAttribute('role', 'button');

// jQuery
$('#foo').removeAttr('role');

// DOM API IE 5.5+
document.getElementById('foo').removeAttribute('role');
~~~

### Adding & Changing Text Content

~~~javascript
// jQuery
$('#foo').text('Goodbye!');

// DOM API, IE 5.5+
document.getElementById('foo').innerHTML = 'Goodbye!';

// DOM API, IE 5.5+ but NOT Firefox
document.getElementById('foo').innterText = 'GoodBye!';

// DOM API, IE 9+
document.getElementById('foo').textContent = 'Goodbye!';
~~~

### Adding/Updating Element Styles

~~~javascript
// jQuery
$('#note').css('fontWeight', 'bold');

// DOM API, IE 5.5+
document.getElementById('note').style.fontWeight = 'bold';
~~~

### Ajax Requests

#### GETting

~~~javascript
// jQuery
$.ajax('myservice/username', {
    data: {
        id: 'some-unique-id'
    }
})
.then(
    function success(name) {
        alert('User\'s name is ' + name);
    },

    function fail(data, status) {
        alert('Request failed.  Returned status of ' + status);
    }
);

// Native XMLHttpRequest Object
var xhr = new XMLHttpRequest();
xhr.open('GET', encodeURI('myservice/username?id=some-unique-id'));
xhr.onload = function(){
  if (xhr.status === 200){
    alert('User\'s name is ' + xhr.responseText);
  }
  else {
    alert('Request failed. Returend status of ' + xhr.status);
  }
};
xhr.send();
~~~

### POSTing

~~~javascript
// jQuery
var newName = 'John Smith';

$.ajax('myservice/username?' + $.param({id: 'some-unique-id'}), {
    method: 'POST',
    data: {
        name: newName
    }
})
.then(
    function success(name) {
        if (name !== newName) {
            alert('Something went wrong.  Name is now ' + name);
        }
    },

    function fail(data, status) {
        alert('Request failed.  Returned status of ' + status);
    }
);


// Native XMLHttpRequest Object
var newName = 'John Smith',
    xhr = new XMLHttpRequest();

xhr.open('POST', encodeURI('myservice/username?id=some-unique-id'));
xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
xhr.onload = function() {
    if (xhr.status === 200 && xhr.responseText !== newName) {
        alert('Something went wrong.  Name is now ' + xhr.responseText);
    }
    else if (xhr.status !== 200) {
        alert('Request failed.  Returned status of ' + xhr.status);
    }
};
xhr.send(encodeURI('name=' + newName));
~~~

### URL Encoding

~~~javascript
// jQuery
$.param({
    key1: 'some value',
    'key 2': 'another value'
});


// Native JavaScript
function param(object) {
    var encodedString = '';
    for (var prop in object) {
        if (object.hasOwnProperty(prop)) {
            if (encodedString.length > 0) {
                encodedString += '&';
            }
            encodedString += encodeURI(prop + '=' + object[prop]);
        }
    }
    return encodedString;
}
~~~

### Sending and Receiving JSON

~~~javascript
// jQuery
$.ajax('myservice/user/1234', {
    method: 'PUT',
    contentType: 'application/json',
    processData: false,
    data: JSON.stringify({
        name: 'John Smith',
        age: 34
    })
})
.then(
    function success(userInfo) {
        // userInfo will be a JavaScript object containing properties such as
        // name, age, address, etc
    }
);

// Web API
var xhr = new XMLHttpRequest();
xhr.open('PUT', 'myservice/user/1234');
xhr.setRequestHeader('Content-Type', 'application/json');
xhr.onload = function() {
    if (xhr.status === 200) {
        var userInfo = JSON.parse(xhr.responseText);
    }
};
xhr.send(JSON.stringify({
    name: 'John Smith',
    age: 34
}));
~~~

### Uploading Files

~~~javascript
// jQuery
var file = $('#test-input')[0].files[0],
    formData = new FormData();

formData.append('file', file);

$.ajax('myserver/uploads', {
    method: 'POST',
    contentType: false,
    processData: false,
    data: formData
});

// jQuery
var file = $('#test-input')[0].files[0];

$.ajax('myserver/uploads', {
    method: 'POST',
    contentType: file.type,
    processData: false,
    data: file
});


// Native JavaScript

var formData = new FormData(),
    file = document.getElementById('test-input').files[0],
    xhr = new XMLHttpRequest();

formData.append('file', file);
xhr.open('POST', 'myserver/uploads');
xhr.send(formData);

var file = document.getElementById('test-input').files[0],
    xhr = new XMLHttpRequest();

xhr.open('POST', 'myserver/uploads');
xhr.setRequestHeader('Content-Type', file.type);
xhr.send(file);
~~~

### CORS, Cross Origin Resource Sharing (sending cross-domain ajax requests)

~~~javascript
// jQuery
$.ajax('http://someotherdomain.com', {
    method: 'POST',
    contentType: 'text/plain',
    data: 'sometext',
    beforeSend: function(xmlHttpRequest) {
        xmlHttpRequest.withCredentials = true;
    }
});

// XMLHttpRequest
var xhr = new XMLHttpRequest();
xhr.open('POST', 'http://someotherdomain.com');
xhr.withCredentials = true;
xhr.setRequestHeader('Content-Type', 'text/plain');
xhr.send('sometext');

// For cross-origin requests, some simple logic
// to determine if XDomainReqeust is needed.
if (new XMLHttpRequest().withCredentials === undefined) {
    var xdr = new XDomainRequest();
    xdr.open('POST', 'http://someotherdomain.com');
    xdr.send('sometext');
}
~~~

### JSONP

~~~javascript
// jQuery
$.ajax('http://jsonp-aware-endpoint.com/user', {
    jsonp: 'callback',
    dataType: 'jsonp',
    data: {
        id: 123
    }
}).then(function(response) {
    // handle requested data from server
});

// Native JavaScript
window.myJsonpCallback = function(data) {
    // handle requested data from server
};

var scriptEl = document.createElement('script');
scriptEl.setAttribute('src',
    'http://jsonp-aware-endpoint.com/user?callback=myJsonpCallback&id=123');
document.body.appendChild(scriptEl);
~~~

## Events

### Sending Native (DOM) Events

~~~javascript
// jQuery
$(anchorElement).click();

// Web API
anchorElement.click();
~~~

### Sending Custom Events

~~~javascript
// jQuery
$('some-element').trigger('my-custom-event');

// Web API
var event = document.createEvent('Event');
event.initEvent('my-custom-event', true, true);
someElement.dispatchEvent(event);

// Web API
var event = new CustomEvent('my-custom-event', {bubbles: true, cancelable: true});
someElement.dispatchEvent(event);
~~~

### Listening For Events

~~~javascript
// jQuery
$(someElement).on('click', function() {
    // TODO event handler logic
});

// jQuery
$(someElement).click(function() {
    // TODO event handler logic
});

// Web API
someElement.addEventListener('click', function() {
    // TODO event handler logic
});
~~~

### Removing Event Handlers

~~~javascript
var myEventHandler = function(event) {
    // handles the event...
}

// jQuery
$('some-element').off('click', myEventHandler);

// Web API
someElement.removeEventListener('click', myEventHandler);
~~~


### Modifying Events

#### Preventing the event from bubbling further up the DOM

~~~javascript
// jQuery
$(someEl).on('some-event', function(event) {
    event.stopPropagation();
});

// Web API
someEl.addEventListener('some-event', function(event) {
    event.stopPropagation();
});
~~~

#### Preventing the event from hitting any additional handlers attached to the current element

~~~javascript
// jQuery
$(someEl).on('some-event', function(event) {
    event.stopImmediatePropagation();
});

// Web API
someEl.addEventListener('some-event', function(event) {
    event.stopImmediatePropagation();
});

~~~

#### Preventing the event from triggering an action defined by the browser

~~~javascript
// jQuery
$(someAnchor).on('click', function(event) {
    event.preventDefault();
});

// Web API
someAnchor.addEventListener('click', function(event) {
    event.preventDefault();
});
~~~

### Event Delegation

~~~javascript
// jQuery
$('#my-list').on('click', 'BUTTON', function() {
    $(this).parent().remove();
});


// Web API
document.getElementById('my-list').addEventListener('click', function(event){
  var clickedEl = event.target;
  if(clickedEl.tagName == 'BUTTON'){
    var listItem = clickedE1.parentNode;
	listItem.parentNode.removeChild(listItem);
  }
});
~~~

### Keyboard Events

~~~javascript
// jQuery
$(document).keydown(function(event) {
  if (event.ctrlKey && event.which == 72) {
    // open help widget
  }
});

// jQuery
$(someElement).keypress(function(event){
  // ...
});

// Web API
document.addEventListener('keydown', function(event) {
  if (event.ctrlKey && event.which === 72){
    // open help widget
  }
});

// Web API
someElement.addEventListener('keypress', function(event){
  // ...
});

// Web API
someElement.addEventListenner('keyup', function(event){
  // ...
});
~~~

### Mouse Events

~~~javascript
// jQuery's hover event
$('some-element').hover(
  function hoverIn(){
    // mouse is hovering over this element
  },

  function hoverOut(){
    // mouse was hovering over this element, but no longer is
  }
);

// Web API
someEl.addEventListener('mouseover', function(){
  // mouse is hovering over this element
});

someEl.addEventListener('mouseout', function(){
  // mouse was hovering over this element, but no longer is
});
~~~

### Browser Load Events

~~~javascript
// jQuery
$(window).load(function(){
  // page is fully rendered
})

// jQuery
$(document).ready(function(){
  // markup is on the page
});

// jQuery
$(function(){
  // markup is on the page
});

// jQuery
$('img').load(function(){
  // image has successfully loaded
})

$('img').error(function(){
  // image has failed to load
})


// Web API
window.addEventListener('load', function(){
  // page is fully rendered
});

// Web API
document.addEventListener('DOMContentLoaded', function(){
  // markup is on the page
});

// Web API
img.addEventListener('load', function(){
  // image has successfully loaded
});

// Web API
img.addEventListenner('error', function(){
  // image has failed to load
});
~~~

### Ancient Browser Support

#### Listening For Events

~~~javascript
someElement.attachEvent('onclick', function(){
  // TODO event handler logic
})

function registerHandler(target, type, callback){
  var listenerMethod = target.addEventListener || target.attachEvent,
    eventName = target.addEventListenner ? type : 'on' + type;
  
  listenerMethod.apply(target, eventName, callback);
}

registerHandler(someElement, 'click', function(){
  // TODO event handler logic
});

function unregisterHandler(target, type, callback){
  var removeMethod = target.removeEventListener || target.detachEvent,
    eventName = target.removeEventListener ? type : 'on' + type;
	removeMethod.apply(target, eventName, callback);
}

unregisterHandler(someElement, 'click', someEventHandlerFunction);
  
~~~

### Form Field Change Events

#### The Event Object

~~~javascript
function myEventHandler(event){
  var actualEvent = event || window.event,
      actualTarget = actualEvent.target || actualEvent.scrElement
}

function myEventHandler(event){
  var actualEvent = event || window.event;

  if (actualEvent.stopPropgation){
    actualEvent.stopPropagation();
  }
  else {
    actualEvent.cancelBubble = true;
  }
}
~~~

## Utilities

### Is this an Object, Array, or a Function?

~~~javascript
// jQuery

$.isFunction(someValue);
$.isPlainObject(someValue);
$.isArray(someValue);

// vanilla JavaScript
// is this a function?
typeof someValue === 'function';

// is this an object?
someValue != null && Object.prototype.toString.call(someValue) === "[object Object]";

// works in modern browsers
Array.isArray(someValue);

// works in all browsers
Object.prototype.toString.call(someValue) === "[object Array]";
~~~

### Combine and copy objects

~~~javascript
var o1 = {
        a: 'a',
        b: {
            b1: 'b1'
        }
    },

    o2 = {
        b: {
            b2: 'b2'
        },
        c: 'c'
    };
	
// jQuery: update o1 with the contents of o2
$.extend(true, o1, o2);


// jQuery: creates a new object that is the aggregate of o1 & o2
var newObj = $.extend(true, {}, o1, o2);

// jQuery: create a copy of one of the objects
var copyOfo1 = $.extend(true, {}, o1);

// vanilla JavaScript
function extend(first, second) {
    for (var secondProp in second) {
        var secondVal = second[secondProp];
        // Is this value an object?  If so, iterate over its properties, copying them over
        if (secondVal && Object.prototype.toString.call(secondVal) === "[object Object]") {
            first[secondProp] = first[secondProp] || {};
            extend(first[secondProp], secondVal);
        }
        else {
            first[secondProp] = secondVal;
        }
    }
    return first;
};

// example use - updates o1 with the content of o2
extend(o1, o2);

// example use - creates a new object that is the aggregate of o1 & o2
var newObj = extend(extend({}, o1), o2);

var copyOfO1 = extend({}, o1);
~~~

### Iterate over object properties

~~~javascript

var parentObject = {
    a: 'a',
    b: 'b'
};

var myObject = Object.create(parentObject);
myObject.c = "c";
myObject.d = "d";


// jQuery
$.each(myObject, function(propName, propValue) {
    // handle each property...
});

// vanilla JavaScript
// works in all browsers
for (var prop in myObject) {
    if (myObject.hasOwnProperty(prop)) {
        // deal w/ each property that belongs only to `myObject`...
    }
}

// works in modern browsers
Object.keys(myObject).forEach(function(prop) {
    // deal with each property that belongs to `myObject`...
});

~~~

### Iterate over array elements

~~~javascript
var myArray = ['a', 'b', 'c'];

// jQuery
$.each(myArray, function(arrayIndex, arrayValue) {
    // handle each array item...
});

// vanilla JavaScript
// works in all browsers
for (var index = 0; i < myArray.length; i++) {
    var arrayVal = myrray[index];
    // handle each array item...
}

// works in modern browsers
myArray.forEach(function(arrayVal, arrayIndex) {
    // handle each array item
}

~~~

### Find an element in an array

~~~javascript
var theArray = ['a', 'b', 'c'];


// jQuery
var indexOfValue = $.inArray('c', theArray);

var allElementsThatMatch = $.grep(theArray, function(theArrayValue) {
    return theArrayValue !== 'b';
});

// vanilla JavaScript

// works in modern browsers
var indexOfValue = theArray.indexOf('c');

// works in all browsers
function indexOf(array, valToFind) {
    var foundIndex = -1;
    for (var index = 0; index < array.length; i++) {
        if (theArray[index] === valToFind) {
            foundIndex = index;
            break;
        }
    }
	return foundIndex;
}

// example use
indexOf(theArray, 'c');


// works in modern browsers
var allElementsThatMatch = theArray.filter(function(theArrayValue) {
    return theArrayValue !== 'b';
});

// works in all browsers
function filter(array, conditionFunction) {
    var validValues = [];
    for (var index = 0; index < array.length; i++) {
        if (conditionFunction(theArray[index])) {
            validValues.push(theArray[index]);
        }
    }
}

// use example
var allElementsThatMatch = filter(theArray, function(arrayVal) {
    return arrayVal !== 'b';
})
~~~

### Turn a pseudo-array into a real array

~~~javascript
var realArray = ['a', 'b', 'c'];

var pseudoArray1 = {
    1: 'a',
    2: 'b',
    3: 'c'
    length: 3
};

// jQuery
var realArray = $.makeArray(pseudoArray);

// vanilla JavaScript
var realArray = [].slice.call(pseudoArray);

[].forEach.call(pseudoArray, function(arrayValue) {
    // handle each element in the pseudo-array...
});

~~~

### Modify a function

#### 1 - Change a function's context

~~~javascript
function Outer() {
    var eventHandler = function(event) {
        this.foo = 'buzz';
    }

    this.foo = 'bar';
    // attach `eventHandler`...
}

var outer = new Outer();
// event is fired, triggering `eventHandler`

// jQuery
function Outer() {
    var eventHandler = $.proxy(function(event) {
        this.foo = 'buzz';
    }, this);

    this.foo = 'bar';
    // attach `eventHandler`...
}

var outer = new Outer();
// event is fired, triggering `eventHandler`

// vanilla JavaScript
// works in modern browsers
function Outer() {
    var eventHandler = function(event) {
        this.foo = 'buzz';
    }.bind(this);

    this.foo = 'bar';
    // attach `eventHandler`...
}

var outer = new Outer();
// event is fired, triggering `eventHandler`

// works in all browsers
function Outer() {
    var self = this,
        eventHandler = function(event) {
            self.foo = 'buzz';
        };

    this.foo = 'bar';
    // attach `eventHandler`...
}

var outer = new Outer();
// event is fired, triggering `eventHandler`
~~~

#### 2 - Create a new function with some pre-determined arguments

~~~javascript
function log(appId, level, message) {
    // send message to central server
}

// jQuery
var ourLog = $.proxy(log, null, 'master-shake');

// example use
ourLog('error', 'dancing is forbidden!');

// vanilla JavaScript
var ourLog = log.bind(null, 'master-shake');

// example use
ourLog('error', 'dancing is forbidden!');

~~~

### Trim a string

~~~javascript

// jQuery
$.trim('  hi there!   ');


// vanilla JavaScript

// works in modern browsers
'  hi there!   '.trim();

// works in all browsers, but needed in IE 8 and older
'  hi there!   '.replace(/^\s+|\s+$/g, '');

// works in all browsers
function trim(string) {
    if (string.trim) {
        return string.trim();
    }
    return string.replace(/^\s+|\s+$/g, '');
}

~~~

### Associate data with an HTML element

~~~html
<div id="one">one</div>
<div id="two">two</div>
~~~

~~~javascript
// jQuery
// make the elements aware of each other
$('#one').data('partnerElement', $('#two'));
$('#two').data('partnerElement', $('#one'));

// on click, either hide or show the partner element
$('#one, #two').click(function() {
    $(this).data('partnerElement').toggle();
});


// vanilla JavaScript

/ works in all browsers
var data = (function() {
    var lastId = 0,
        store = {};

    return {
        set: function(element, info) {
            var id;
            if (element.myCustomDataTag === undefined) {
                id = lastId++;
                element.myCustomDataTag = id;
            }
            store[id] = info;
        },

        get: function(element) {
            return store[element.myCustomDataTag];
        }
    };
}());

// make the elements aware of each other
var one = document.getElementById('one'),
    two = document.getElementById('two'),
    toggle = function(element) {
        if (element.style.display !== 'none') {
            element.style.display = 'none';
        }
        else {
            element.style.display = 'block';
        }
    };

data.set(one, {partnerElement: two});
data.set(two, {partnerElement: one});

// on click, either hide or show the partner element
// remember to use `attachEvent` in IE 8 and older, if support is required
one.addEventListener('click', function() {
    toggle(data.get(one).partnerElement);
});
two.addEventListener('click', function() {
    toggle(data.get(two).partnerElement);
});


// works only in the latest browsers
// make the elements aware of each other
var weakMap = new WeakMap(),
    one = document.getElementById('one'),
    two = document.getElementById('two'),
    toggle = function(element) {
        if (element.style.display !== 'none') {
            element.style.display = 'none';
        }
        else {
            element.style.display = 'block';
        }
    };

weakMap.set(one, {partnerElement: two});
weakMap.set(two, {partnerElement: one});

// on click, either hide or show the partner element
// remember to use `attachEvent` in IE 8 and older, if support is required
one.addEventListener('click', function() {
    toggle(weakMap.get(one).partnerElement);
});
two.addEventListener('click', function() {
    toggle(weakMap.get(two).partnerElement);
});

// works in all browsers
var data = window.WeakMap ? new WeakMap() : (function() {
    var lastId = 0,
        store = {};

    return {
        set: function(element, info) {
            var id;
            if (element.myCustomDataTag === undefined) {
                id = lastId++;
                element.myCustomDataTag = id;
            }
            store[id] = info;
        },

        get: function(element) {
            return store[element.myCustomDataTag];
        }
    };
}());

~~~


