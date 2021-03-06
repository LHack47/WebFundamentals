---
layout: shared/narrow
title: "Optimize JavaScript execution"
description: "JavaScript is often the trigger for visual changes. Sometimes that's directly through style manipulations, and sometimes it's calculations that will result in visual changes, like searching or sorting some data. Badly-timed or long-running JavaScript can be a common cause of performance issues, and you should look to minimize its impact where you can."
published_on: 2015-03-20
updated_on: 2015-03-20
order: 2
translation_priority: 0
authors:
  - paullewis
notes:
  jit:
    - "If you really want to see JIT in action you should check out <a href='http://mrale.ph/irhydra/2/'>IRHydra<sup>2</sup> by Vyacheslav Egorov</a>. It shows the intermediate state of JavaScript code when Chrome’s JavaScript engine, V8, is optimizing it."
key-takeaways:
  - "Avoid setTimeout or setInterval for visual updates; always use requestAnimationFrame instead."
  - "Move long-running JavaScript off the main thread to Web Workers."
  - "Use micro-tasks to make DOM changes over several frames."
  - "Use Chrome DevTools’ Timeline and JavaScript Profiler to assess the impact of JavaScript."
---

<p class="intro">
  JavaScript is often the trigger for visual changes. Sometimes that's 
  directly through style manipulations, and sometimes it's calculations that 
  will result in visual changes, like searching or sorting some data. 
  Badly-timed or long-running JavaScript can be a common cause of performance 
  issues, and you should look to minimize its impact where you can.
</p>

{% include shared/toc.liquid %}

{% include shared/takeaway.liquid list=page.key-takeaways %}

JavaScript performance profiling can be something of an art, because the JavaScript you write is nothing like the code that is actually executed. Modern browsers use JIT compilers and all manner of optimizations and tricks to try and give you the fastest possible execution, and this substantially changes the dynamics of the code.

{% include shared/remember.liquid title="Note" list=page.notes.jit %}

With all that said, however, there are some things you can definitely do to help your apps execute JavaScript well.

## Use `requestAnimationFrame` for visual changes

When visual changes are happening on screen you want to do your work at the right time for the browser, which is right at the start of the frame. The only way to guarantee that your JavaScript will run at the start of a frame is to use `requestAnimationFrame`.

{% highlight javascript %}
/**
 * If run as a requestAnimationFrame callback, this
 * will be run at the start of the frame.
 */
function updateScreen(time) {
  // Make visual updates here.
}

requestAnimationFrame(updateScreen);
{% endhighlight %}

Frameworks or samples may use `setTimeout` or `setInterval` to do visual changes like animations, but the problem with this is that the callback will run at _some point_ in the frame, possibly right at the end, and that can often have the effect of causing us to miss a frame, resulting in jank.

<img src="images/optimize-javascript-execution/settimeout.jpg" alt="setTimeout causing the browser to miss a frame.">

In fact, jQuery’s default `animate` behavior today is to use `setTimeout`! You can [patch it to use `requestAnimationFrame`](https://github.com/gnarf/jquery-requestAnimationFrame), which is strongly advised.

## Reduce complexity or use Web Workers

JavaScript runs on the browser’s main thread, right alongside style calculations, layout, and, in many cases, paint. If your JavaScript runs for a long time, it will block these other tasks, potentially causing frames to be missed.

You should be tactical about when JavaScript runs, and for how long. For example, if you’re in an animation like scrolling, you should ideally be looking to keep your JavaScript to something in the region of **3-4ms**. Any longer than that and you risk taking up too much time. If you’re in an idle period, you can afford to be more relaxed about the time taken.

In many cases you can move pure computational work to [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/basic_usage), if, for example, the it doesn’t require DOM access. Data manipulation or traversal, like sorting or searching, are often good fits for this model, as are loading and model generation.

{% highlight javascript %}
var dataSortWorker = new Worker("sort-worker.js");
dataSortWorker.postMesssage(dataToSort);

// The main thread is now free to continue working on other things...

dataSortWorker.addEventListener('message', function(evt) {
   var sortedData = evt.data;
   // Update data on screen...
});

{% endhighlight %}

Not all work can fit this model: Web Workers do not have DOM access. Where your work must be on the main thread, consider a batching approach, where you segment the larger task into micro-tasks, each taking no longer than a few milliseconds, and run inside of `requestAnimationFrame` handlers across each frame.

{% highlight javascript %}
var taskList = breakBigTaskIntoMicroTasks(monsterTaskList);
requestAnimationFrame(processTaskList);

function processTaskList(taskStartTime) {
  var taskFinishTime;

  do {
    // Assume the next task is pushed onto a stack.
    var nextTask = taskList.pop();

    // Process nextTask.
    processTask(nextTask);

    // Go again if there’s enough time to do the next task.
    taskFinishTime = window.performance.now();
  } while (taskFinishTime - taskStartTime < 3);

  if (taskList.length > 0)
    requestAnimationFrame(processTaskList);

}
{% endhighlight %}

There are UX and UI consequences to this approach, and you will need to ensure that the user knows that a task is being processed, either by [using a progress or activity indicator](https://www.google.com/design/spec/components/progress-activity.html). In any case this approach will keep your app's main thread free, helping it to stay responsive to user interactions.

## Know your JavaScript’s “frame tax”

When assessing a framework, library, or your own code, it’s important to assess how much it costs to run the JavaScript code on a frame-by-frame basis. This is especially important when doing performance-critical animation work like transitioning or scrolling.

The best way to measure your JavaScript’s cost and performance profile is to use Chrome DevTools. Typically you will get low detail records that look like this:

<img src="images/optimize-javascript-execution/low-js-detail.jpg" alt="Chrome DevTools' Timeline providing low JS execution detail.">

If you discover that you have long-running JavaScript, you can enable the JavaScript profiler at the top of the DevTools user interface:

<img src="images/optimize-javascript-execution/js-profiler-toggle.jpg" alt="Enabling the JS profiler in DevTools.">

There is an overhead to profiling JavaScript in this way, so be sure to only enable it when you want more insight into JavaScript runtime characteristics. With the checkbox enabled you can now perform the same actions and you’ll get significantly more information on which functions were called in your JavaScript:

<img src="images/optimize-javascript-execution/high-js-detail.jpg" alt="Chrome DevTools' Timeline providing high JS execution detail.">

Armed with this information you can assess the performance impact of the JavaScript on your application, and begin to find and fix any hotspots where functions are taking too long to execute. As mentioned earlier you should seek to either remove long-running JavaScript, or, if that’s not possible, move it to a Web Worker freeing up the main thread to continue on with other tasks.

## Avoid micro-optimizing your JavaScript

It may be cool to know that the browser can execute one version of a thing 100 times faster than another thing, like that requesting an element’s `offsetTop` is faster than computing `getBoundingClientRect()`, but it’s almost always true that you’ll only be calling functions like these a small number of times per frame, so it’s normally wasted effort to focus on this aspect of JavaScript’s performance. You'll typically only save fractions of milliseconds.

If you’re making a game, or a computationally expensive application, then you’re likely an exception to this guidance, as you’ll be typically fitting a lot of computation into a single frame, and in that case everything helps.

In short, you should be very wary of micro-optimizations because they won’t typically map to the kind of application you’re building.
