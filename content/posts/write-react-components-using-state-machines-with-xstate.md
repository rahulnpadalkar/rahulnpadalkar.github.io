---
title: "Write React Components Using State Machines With Xstate"
date: 2022-03-17T21:14:56+05:30
draft: false
---

I have been writing Javascript for over 4 years now. I have spent a major chunk of it developing frontend applications mostly in React. I have spent enough sleepless nights debugging React components to realize the importance of state in a declarative UI framework like React. Though we have a lot of libraries which help us manage state, none assist in writing components with state as the protagonist.

Recently i came across XState which is a library to create state machines in Javascript. I have been playing around with it for some time now. In this post, I will take you through a very basic example of a toggle button that you can implement using XState in React.

Before we jump into how to use state machines in React, let us first take a brief look at what state machines are.

If you wish to directly jump and look at the code [click here](https://tabsoverspaces.in/posts/write-react-components-using-state-machines-with-xstate/#designing-a-content-toggle-button).

## What are state machines?

According to [Wikipedia](https://en.wikipedia.org/wiki/Finite-state_machine),

> A **finite-state machine** or simply a **state machine** is a mathematical model of computation. It is an abstract machine that can be in exactly one of a finite number of states at any given time.

Simply put, a state machine is composed of states and transitions between those states. Diagrammatically, it is a bunch of circles representing states connected by arrows that show the transition into different states when passed an input value.

The most simple state machine I can think of right now is a toggle button.

![Simple State Machine](/images/small_simple.png)

In this state machine, there are,

2 states →

- On
- Off

2 State transitions →

- When in “On” state and on “Click” input transition to “Off” state
- When in “Off” state and on “Click” input transition to “On” state

The key here is to identify states a machine (in our case React components) can be in and on what inputs (in our case events) does it transition to those states.

Now let’s take a look at how we use state machines in React.

## Designing a content toggle button

The requirement for a content toggle button is simple. On clicking, it should either hide or show content. Let’s first take a look at how the state machine for this would look.

![Simple Toggle Machine](/images/small_toggle.png)

Here we have 2 states, “Hide” and “Show”. A “click” event that toggles the machine between the 2 states. Pretty simple and straightforward.

Now let’s create a simple machine using XState

```jsx
import { createMachine } from "xstate";

const toggleMachine = createMachine({
  // content is visible initially
  initial: "show",
  // defining the 2 states
  states: {
    hide: {
      on: {
        // defining what happens on click when in hide state
        click: {
          target: "show",
        },
      },
    },
    show: {
      on: {
        // defining what happens on click when in show state
        click: {
          target: "hide",
        },
      },
    },
  },
});
```

Here we define the 2 states (ie. show and hide) and the transitions and pass them to the `createMachine` function from `xstate`. One thing to note here is the event name (here it is `click`) can be anything. It doesn’t have to be a browser-defined event. That’s because XState is designed to be platform and framework agnostic (more on this later).

Now let’s use this machine in React.

```jsx
import { useMachine } from "@xstate/react";

export default function ContentToggle() {
  const [state, send] = useMachine(toggleMachine);

  return (
    <div className="App">
      <button onClick={() => send("click")}>Toggle Content</button>
      {state.value === "show" ? (
        <div
          style={{ padding: "2rem", margin: "2rem", border: "2px solid red" }}
        >
          This content toggles
        </div>
      ) : (
        ""
      )}
    </div>
  );
}
```

XState has React utility library ([xstate/react](https://github.com/statelyai/xstate/tree/main/packages/xstate-react)) which supports hooks for using state machines created using XState. Here we use the `useMachine` hook from the library to use the `toggleMachine` we defined earlier.

The hook returns 2 things, a function `send` and the current state `state`. The send function can be used to send “input” to the state machine. The `onClick` handler sends a `click` input to the state machine. The state machine then transitions into the appropriate state and is reflected in the `state` value returned by the `useMachine` hook.

Pretty cool imo!

Demo Sandbox → [https://codesandbox.io/s/xstate-demo-j41ody?file=/src/App.js:63-106](https://codesandbox.io/s/xstate-demo-j41ody?file=/src/App.js:63-106)

## Yet another state management library?

When I first started using XState I was confused if I should use XState as a state management library. So I asked [David](https://twitter.com/DavidKPiano) (creator of XState) about this on [one of the Learn With Jason live streams](https://www.youtube.com/watch?v=RYQ_HG7vVZw&t=1380s).

> **In summary, XState is a state orchestration library rather than a state management library.**

So it should be used along with something like redux or recoil. XState is more like a life coach who can answer the “What do I do when \<event\> happens?” type of questions. Though XState is a state orchestration library it can still hold data and can be used for state management.

## XState is cool!

State is at the core of React, or for that matter any declarative UI library. There are myriad state management libraries (I don’t know why React doesn’t ship with one, personal gripe) for React but we don’t have any tool to better visualize and understand all the states and their transitions. XState solves the latter and does it in a pretty clean way.

### 👀 Visualize states and transitions

The cool thing about XState is the ability to drop a state machine in their [visualizer](https://xstate.js.org/viz/) and _voila!_

![machine](/images/machine.png#center)

The visualizer does a fantastic job of showing all the states and transitions. The really cool part is, it is interactive, meaning you can click on the events (`click` in this case) which would then highlight the next state and all the events that can happen when in that state.

You can try it yourself by pasting the code below in the editor [on this page](https://xstate.js.org/viz/).

```jsx
const toggleMachine = Machine({
  initial: "show",
  states: {
    hide: {
      on: {
        click: {
          target: "show",
        },
      },
    },
    show: {
      on: {
        click: {
          target: "hide",
        },
      },
    },
  },
});
```

Pretty cool right?

### 🧘 Framework Agnostic

State machines are a concept in computer science and are used universally. XState helps in creating state machines using javascript. State machines created using XState don’t care where they are used, meaning they are platform and framework agnostic. So you can rip out a state machine from a React codebase and use it in Vue or if you are adventurous then even in jQuery or Vanilla JS. You can also use it in a NodeJS backend. I find this extremely useful.

A marketplace for state machines is a really cool product idea btw.

### 📝 Self Documenting

We all know how much developers like to document their code. I remember in a conversation with a super experienced developer, he told me code is the best documentation. I mean you can have confluence pages and comments in the code, but the most updated version of documentation will always be the code. The way we define state machines in XState, make it very easy to read and understand.

I can see that once the component grows in complexity, it can be a bit difficult but you can always use the visualizer.

### 🧗‍♂️ Edge cases

Designing a state machine for a component forces you to think about all the states a component can be in. It also forces you to think about how does the component transition from one state to another. I think that paints a very clear picture of how the component will work. It helps in identifying edge cases early on and reduces the chances of the component transitioning into an unpredictable state.

That’s it!

If you liked this blog post you might also like,

👉 [Writing a useState hook from scratch](https://tabsoverspaces.in/posts/writing-use-state-hook-from-scratch/)

👉 [OAuth2 with Reddit API](https://tabsoverspaces.in/posts/oauth2-with-reddit-api/)

I am on twitter as [rahulnpadalkar](https://twitter.com/rahulnpadalkar). I sometimes post memes and hot takes there.

I also make [Youtube videos](https://www.youtube.com/channel/UCrhExmTHdRNFDRlg7wz_UYA).

Some of my most watched videos.

🔗 [Event Delegation in Javascript](https://www.youtube.com/watch?v=yn_3MncFMJc)

🔗 [Writing a custom middleware in Express](https://www.youtube.com/watch?v=Brr721Du8U4)

🔗 [Writing videos in React with Remotion](https://youtu.be/in2q8Xfmk2E)
