---
title: "Writing a custom Supabase Hook"
date: 2022-02-05T18:36:20+05:30
draft: false
---

Hooks in React were introduced in React 16.8. Hooks became a hit and saw a lot of adoption. A lot of libraries in the ecosystem now provide hooks support. One of the main motivations for introducing hooks was to enable sharing of state logic across components. You can read more [about this here](https://reactjs.org/docs/hooks-intro.html).

In this post, I will go over how to write a custom hook with an example.

## Problem

I was working with Supabase for one of my side projects and I wanted to abstract away the API calls made to Supabase from components. So I thought this would be a good opportunity to write a custom hook.

Here’s a list of requirements for the custom hook:

- Send a request to the supabase endpoint.
- Handle errors if any.
- Return data from the request.

Pretty simple.

## Solution

```jsx
//useSupabase.js

import { useState, useEffect } from "react";

//supaCall calls the Supabase API
const useSupabase = (supaCall) => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [data, setData] = useState(null);

  useEffect(() => {
    async function getData() {
      setLoading(true);
      try {
        let { data, error } = await supaCall();
        if (error) {
          setError(error);
          // Show toast with relevant error message to user
          // Log error to Airbrake or Sentry
        } else {
          setData(data);
        }
      } catch (e) {
        // Likely to be a network error
        setError(e);
      } finally {
        setLoading(false);
      }
    }
    getData();
  }, []);

  return { loading, data, error };
};

export default useSupabase;
```

`useSupabase` is a function that accepts a function as a parameter.

There are 3 state variables →

- loading → Describes the state of the request. This can be used to show a loading icon.
- error → captures errors if any.
- data → Raw data returned by the API call.

### Things to note

- Global error handling can be done after or before the `setError` call. The error can be logged to Airbrake or similar tools. For better UX, a toast message with a relevant error message can be shown to the user.
- The `supaCall` argument passed to the hook is a function. More on that later.
- The Supabase API is called in the `useEffect` hook. So whenever the component (which uses this hook) is mounted the API call is triggered.

## Usage

Let’s quickly take a look at how the custom hook is used.

```jsx
//Posts.js
import useSupabase from "../hooks/useSupabase";
import { getAllPosts } from "../services/supaservice";

function Posts() {
  const { loading, data, error } = useSupabase(getAllPosts);

  return (
    <Flex flexDirection="column" mt="10">
      {loading && <Text>Loading...</Text>}
      {!loading && error && <Text>Error</Text>}
      {!loading && data && data.map((someProps) => <Post {...someProps} />)}
    </Flex>
  );
}

// services/supaservice.js
const getAllPosts = async () => {
  let { data, error } = await supabase.from("posts").select("*");
  return { data, error };
};
```

The `getAllPosts` method is passed as an argument to the `useSupabase` hook. The `getAllPosts` function calls the Supabase API (using the supabase client).

So the way it works is, the `useSupabase` hook gets called with the `getAllPosts` method, and when the component is mounted, the API call is triggered which returns all posts and gets rendered on the screen.

### Usage with a parameter passed to the function

In many cases, the function passed to `useSupabase` would require an input. In that case, we can use bind and pass arguments.

```jsx
import { getDisplayName } from "../services/supaservice";

function Profile({ id }) {
  const { loading, data, error } = useSupabase(getDisplayName.bind(this, id));

  return <Flex>{!loading && data && <Text>{data["display_name"]}</Text>}</Flex>;
}

export default Profile;

// services/supabaseservice.js

const getDisplayName = async (id) => {
  const { data, error } = await supabase
    .from("profiles")
    .select("display_name");
  return { data, error };
};
```

## But wait..this isn’t good enough

There is a problem here though. There are cases where the API needs to be called after the user performs an action. For example, adding a new post after the user clicks on the submit button. For those cases, this hook isn’t suitable.

## useSupabase with a callback

```jsx
import { useState, useEffect } from "react";

const useSuapbaseWithCallback = (supaCall) => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [data, setData] = useState(null);

  async function callService() {
    setLoading(true);
    try {
      let { data, error } = await supaCall(arguments);
      if (error) {
        setError(error);
      } else {
        setData(data);
      }
    } catch {
      setError(error);
    } finally {
      setLoading(false);
    }
  }
  return { callService, loading, data, error };
};

export default useSuapbaseWithCallback;
```

The change is subtle but important. Rather than the API getting called on the component mount, a callback is returned as one for the return values. This callback can then be used to call the API in event handlers.

In the code above, `callService` is returned as a callback. This is similar to the `useMutation` hook from the [Apollo GraphQL library](https://www.apollographql.com/docs/react/api/react/hooks/#usemutation).

## Usage (useSupabaseWithCallback)

```jsx
import { updateDisplayName } from "../services/supaservice";
import useSuapbaseWithCallback from "../hooks/useSuapbaseWithCallback";

function Account() {
  const [newDisplayName, setNewDisplayName] = useState(null);

  const { callService, loading, data, error } =
    useSuapbaseWithCallback(updateDisplayName);

  const updateName = async () => {
    if (newDisplayName) {
      await callService(newDisplayName, user.id);
    }
  };

  return (
    <Flex flexDirection="column">
      .....
      <Button onClick={updateName}>Update</Button>
      ....
    </Flex>
  );
}

export default Account;

// services/supabaseservice.js

const updateDisplayName = async ([display_name, id]) => {
  const { data, error } = await supabase
    .from("profiles")
    .update({ display_name })
    .match({ id });
  return { data, error };
};
```

The `callService` is a callback returned by `useSuapbaseWithCallback` hook. This callback can be called after a user performs a certain action. In the code above, the callback is called after the user clicks on the update button.

And that’s how you can write custom hooks in react. Remember to start the name of your custom hook with **use** so that it gets marked if its usage violates the rules for hooks.

If you liked this blog post you might also like,

👉 [Writing a useState hook from scratch](https://tabsoverspaces.in/posts/writing-use-state-hook-from-scratch/)

👉 [OAuth2 with Reddit API](https://tabsoverspaces.in/posts/oauth2-with-reddit-api/)

I am on twitter as [rahulnpadalkar](https://twitter.com/rahulnpadalkar). I sometimes post memes and hot takes there.

I also make [Youtube videos](https://www.youtube.com/channel/UCrhExmTHdRNFDRlg7wz_UYA).

Some of my most watched videos

🔗 [Event Delegation in Javascript](https://www.youtube.com/watch?v=yn_3MncFMJc)

🔗 [Writing a custom middleware in Express](https://www.youtube.com/watch?v=Brr721Du8U4)
