# shouldYield

## The Problem
Today, developers must tradeoff how fast they can accomplish large blocks of
work and how responsive they are to user input. For example, during page load,
there may be a set of components and scripts to initialize. These are often
ordered by priority: for example, first installing event handlers on primary
buttons, then a search box, then a messaging widget, and then finally moving on
to analytics scripts and ads.

In this example, a developer can either optimize for:
* Completing all work as fast as possible.
  * For example, we want the messaging widget to be initialized by the time the
    user interacts with it.
* Responding to user input as fast as possible.
  * For example, when the user taps one of our primary buttons, we don't want to
    block until our whole page is ready to go before responding.

Completing all work as fast as possible is easy, the developer can simply
execute all the work in the minimal number of tasks.

Responding to user input as fast as possible today requires paying some
performance overhead, and minimizing that performance overhead is quite
complicated. The most straight forward approach is to perform a unit of work,
and then `setTimeout(..., 0)` or `setImmediate(...)` to continue the work.

The performance overhead of this approach comes from a few sources:
* Inherent overhead in posting a task.
* Task throttling from the browser.
* Execution of other browser work, such as rendering, postponing task execution.
  * Sometimes this is desirable, to get something visible on screen sooner. If
    we're executing script which is required to display our app however, this
    isn't desirable.

## Proposal

In order to enable developers to complete their work as fast as possible if the
user isn't interacting, but respond to user input as fast as possible if input
occurs, we propose adding a new `window.shouldYield()` API, which returns true
if there is user input pending. To avoid script from misbehaving, and preventing rendering for multiple seconds, user agents will also be permitted to return `true` if script has been executing for too long.

## Example

Using `shouldYield` requires having some way to schedule tasks. We anticipate
most adoption coming from frameworks and large sites. However, if you have a
list of tasks that need executing, adoption is very simple.

```javascript
let taskQueue = [task1, task2, ...];

function doWork() {
  while (let task = taskQueue.pop()) {
    task.execute();
    if (shouldYield()) {
      setTimeout(doWork, 0);
      break;
    }
  }
}

doWork();
```

## Constraints

Ideally, a solution to this problem would meet the following constraints:
1. Enable stopping JS execution when input arrives.
1. Enable efficiently resuming JS execution once input has been processed.
1. Provide the browser a way to stop the JS from executing when input isn’t
   pending, in case the browser decides it really needs to draw something.
1. Not require paying the cost of posting a task unless the browser has
   pending high priority work
1. Prevent other JS from running between when the script stops and resumes JS
   execution
   * This excludes JS associated with browser work like event processing, rAF
     callbacks etc

This proposal focuses on constraints 1, 3 & 4, and ignores 2 and 5, which will
be addressed independently.

The fifth constraint is interesting - in order for work which is incentivized to
finish quickly (e.g., ads script) to be able to adopt this API and improve
responsiveness to input, we need some way to prevent arbitrary javascript from
executing between when script yields and when it is rescheduled.
