---
title: "Oauth2 With Reddit Api"
date: 2022-01-23T14:55:22+05:30
draft: false
---

**Read time ~ 3 coffees ☕☕☕** 

I recently started working on a project which allows Reddit users to schedule posts for multiple subreddits. To post something on behalf of the user, many services support the [OAuth2](https://oauth.net/2/) protocol for authorization. Reddit does too. Reddit supports 3 types of applications under this flow. **The one we are interested in is the web app one**. Refer to [this doc](https://github.com/reddit-archive/reddit/wiki/oauth2-app-types) if you are confused about choosing the right type.

# Create an application

> *If you have created an application in Reddit, then you can skip this step.*
> 

Head over to [https://www.reddit.com/prefs/apps](https://www.reddit.com/prefs/apps)

![new_app_create.jpg](OAuth2%20with%20Reddit%20API%20dd5eff4ad2a5443088696390878c35a6/new_app_create.jpg)

Click on the button to create a new app.

![Screenshot 2022-01-20 at 23-41-13 preferences (reddit com).png](OAuth2%20with%20Reddit%20API%20dd5eff4ad2a5443088696390878c35a6/Screenshot_2022-01-20_at_23-41-13_preferences_(reddit_com).png)

Fill in the details and hit create an app. Few things to note.

1. The app is of type **web type**.
2. Redirect URI is important so please make a note of it before saving. I generally put something like `http://localhost:3000/login/callback` there.

Congrats, you successfully created an application. 🎉

Before we move on let’s first try to understand the flow. This will help us understand the significance of `redirect_uri`

# The OAuth Flow

![Rectangle 1.png](OAuth2%20with%20Reddit%20API%20dd5eff4ad2a5443088696390878c35a6/Rectangle_1.png)

The diagram above describes the OAuth flow. Let’s go over the flow once and then dive into the code.

1. The user visits the Web Application and clicks on the “Login With Reddit” button.
2. The user is then taken to Reddit and is asked to either deny or approve the permissions requested by the application.
3. If the user approves, then the user is redirected to the `redirect_uri` (Remember this?). This will redirect the user back to the web application.
4. The `redirect_uri` redirects the user back to the web app along with 2 things. `code` and `state` . We will talk about `state` later. 
5. The web app then makes requests to the Reddit API for the user’s `access_token`. The `code` value received in the previous step is passed as a parameter in this request.
6. Reddit validates the `code` passed and then returns an `access_token` and a `refresh_token`.
7. The web app saves the `access_token` and now can make requests to the Reddit API on behalf of the user.

> Some *OAuth providers return an access_token after step 3. But for Reddit, we need to perform a few more steps.*
> 

 

# The Code.

```jsx
function Home(props) {

	function openLogin() {
        window.open(
            `https://www.reddit.com/api/v1/authorize?
							client_id=YOUR_CLIENT_ID&response_type=code
							&state=random-string
							&redirect_uri=http://localhost:3000/login/callback
							&duration=permanent&scope=identity,submit,save`,
            "_self"
            )
    }

	return (
		...
		<Button onClick={() => {openLogin()}}>Login With Reddit</Button>
		....
	)
}
```

This is a react component. But it can be anything. The important thing is the user should be redirected to Reddit after clicking the button.

### A few things to note.

- Here, we redirect the user to the  `/authorize` endpoint. We pass a few query parameters. Let’s go over them quickly.
    - `client_id` This is the string that you can see below the name of the application we [created in step 1](https://www.notion.so/OAuth2-with-Reddit-API-8c239f5364f0458d91367edfd7a45ad0).
    - `response_type` This needs to be set to `code` You can read more on this [here](https://github.com/reddit-archive/reddit/wiki/OAuth2#authorization).
    - `state` This is a random string that you pass to the endpoint. This same string is returned later when Reddit redirects the user to the `redirect_uri`. This field is important to understand if the web app has received a response to a request that it initiated.
    - `redirect_uri` This should match with the one in step 1.
    - `duration` This defines for how long do you need the `access_token`. All `access_token` expire after 1 hour. More on this [here](https://github.com/reddit-archive/reddit/wiki/OAuth2#authorization).
    - `scope` This defines the permissions the web app needs, to function properly. A list along with their description of each scope is available here [https://www.reddit.com/api/v1/scopes](https://www.reddit.com/api/v1/scopes)

The user will be presented with the permission screen like below if everything goes right.

![permission.jpg](OAuth2%20with%20Reddit%20API%20dd5eff4ad2a5443088696390878c35a6/permission.jpg)

Users can either **Allow** or **Decline** access to the web application.

Let’s consider the happy path ie. user allows access. **In both cases, the user is redirected to `redirect_uri`**

```jsx
function LoginWithReddit({location}) {
    let {search} = useLocation()

    useEffect(() => {
        const query = new URLSearchParams(search);
        axios.get(`http://localhost:3003/login_reddit?code=${query.get("code")}`)
    })

    return (
        <Flex>
            <CircularProgress isIndeterminate color="orange"/>
        </Flex>
    )
}

// This is how react router is setup.

<BrowserRouter>
    <Routes>
      <Route path="/login/callback" element={<LoginWithReddit/>} />
      <Route path="/" element={<Home/>} />
    </Routes>
</BrowserRouter>
```

This is the react component that gets rendered when the user is redirected back to the web app.

### Things to note

- The `code` received in the query parameter is sent to the back end. The back end then requests the Reddit API to get the necessary tokens.
- An alternative approach could be to redirect the user to the back-end directly, which will save an API call.

## Back-end server code

```jsx
... // express server boilerplate

app.get("/login_reddit", async(req,res) => {
    try {
				// GET USER ACCESS TOKEN
        const code = req.query.code;
        const encodedHeader = Buffer.from(`${process.env.REDDIT_CLIENT_ID}:${process.env.REDDIT_CLIENT_SECRET}`).toString("base64")
        let response = await fetch(`https://www.reddit.com/api/v1/access_token`,  {
            method: 'POST',
            body: `grant_type=authorization_code&code=${code}&redirect_uri=http://localhost:3000/login/callback`,
            headers: {authorization: `Basic ${encodedHeader}`, 'Content-Type': 'application/x-www-form-urlencoded'}
        });
        let body = await response.json();

				// GET USER INFORMATION
        response = await fetch(`https://oauth.reddit.com/api/v1/me`, {
            method: "GET",
            headers: {authorization: `bearer ${body.access_token}`}
        })
        let user = await response.json();
        await signInUser(user['id'],user['name'],body['access_token'],body['refresh_token'])
        let token = await signJWT(user['id'])
        res.send({token})
    } catch(e) {
        res.send({msg: "Something went wrong."})
    }
})

... // other endpoints
```

This is a code snippet from an express server that exposes `GET /login_reddit` endpoint.

### Things to note

**Getting user tokens `POST /api/v1/access_token`**

- Reddit uses [HTTP Basic Authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication) for fetching user access and refresh tokens. This authentication mechanism requires that the client sends base64 encoded credentials in the `Authorization` header in `Basic [base64 encoded credentials here]` format.
- 3 things need to be passed in the body here
    - `grant_type` This needs to be set to `authorization_code`
    - `code` This needs to be set to the value returned by Reddit after redirecting the user to the `redirect_uri`
    - `redirect_uri` I don’t know why we need to pass this again 🤷🏻
- Along with the `Authorization` header, the `Content-Type` header needs to be set to `application/x-www-form-urlencoded` This is super important. You might also need to set `User-Agent`. This is generally the name of your application.
- The response finally returns the tokens. An `access_token` and a `refresh_token`. Access tokens are valid only for 1 hour. To get a fresh `access_token`,  `refresh_token` is used.

**Getting User Information  `GET api/v1/me`**

- To get user information, the request needs to be made to Reddit OAuth URL. So instead of `https://www.reddit.com` request needs to be sent to `https://oauth.reddit.com` **Please make sure you are sending all your requests (for protected endpoints) to this URL. Otherwise, you will end up with 401 Unauthorized responses.**
- The `Authorization` header is set to the `access_token` that was sent in the above step. Reddit follows the bearer schema which means the header should be set to a value of the format `bearer [access_token_here]`

**Logging in the user**

- Once the user information is fetched from Reddit. A JWT can be created and sent back to the browser which will then log in the user into the web application.


> ‼️ JWT isn’t secure. It is just a base64 encoded string. Storing sensitive information like passwords in these tokens could pose a huge security risk.


To re-issue access tokens after they have expired can be done by calling `/api/access_token` It is pretty straight forward and more information can be found [here](https://github.com/reddit-archive/reddit/wiki/OAuth2#refreshing-the-token). There can be a few strategies to get fresh access tokens.

- A cron job can be set up that checks and fetches fresh access tokens for the ones that have expired. The downside I can see to this approach is this will fetch fresh access tokens for accounts that are inactive thus wasting precious API requests. Also as the app scales, a single cron job might not be sufficient.
- Fresh access tokens can be fetched on-demand. If Reddit API returns an expired access token error then a fresh access token can be fetched. This ensures that requests are made only for active users.

And that’s it. The web app can now call protected Reddit API endpoints on behalf of the user.

Comment if you have any questions. You can follow me on [Twitter](https://twitter.com/rahulnpadalkar). I usually tweet about JavaScript. 

If you are learning React then you might be interested in [Writing useState hook from scratch.](https://www.notion.so/Writing-useState-hook-from-scratch-4a633e81e7634b0ea351256f03340581) 

I also make [YouTube](https://www.youtube.com/channel/UCrhExmTHdRNFDRlg7wz_UYA) videos. Go, check them out!

