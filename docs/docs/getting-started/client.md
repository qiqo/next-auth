---
id: client
title: Client API
---

The NextAuth.js client library makes it easy to interact with sessions from React applications.

#### Example Session Object

```ts
{
  user: {
    name: string
    email: string
    image: string
  },
  expires: Date // This is the expiry of the session, not any of the tokens within the session
}
```

:::tip
The session data returned to the client does not contain sensitive information such as the Session Token or OAuth tokens. It contains a minimal payload that includes enough data needed to display information on a page about the user who is signed in for presentation purposes (e.g name, email, image).

You can use the [session callback](/configuration/callbacks#session-callback) to customize the session object returned to the client if you need to return additional data in the session object.
:::

:::note
The `expires` value is rotated, meaning whenever the session is retrieved from the [REST API](/getting-started/rest-api), this value will be updated as well, to avoid session expiry.
:::

---

## useSession()

- Client Side: **Yes**
- Server Side: No

The `useSession()` React Hook in the NextAuth.js client is the easiest way to check if someone is signed in.

Make sure that [`<SessionProvider>`](#sessionprovider) is added to `pages/_app.js`.

#### Example

```jsx
import { useSession } from "next-auth/react"

export default function Component() {
  const { data: session, status } = useSession()

  if (status === "authenticated") {
    return <p>Signed in as {session.user.email}</p>
  }

  return <a href="/api/auth/signin">Sign in</a>
}
```

`useSession()` returns an object containing two values: `data` and `status`:

- **`data`**: This can be three values: [`Session`](https://github.com/nextauthjs/next-auth/blob/8ff4b260143458c5d8a16b80b11d1b93baa0690f/types/index.d.ts#L437-L444) / `undefined` / `null`.
  - when the session hasn't been fetched yet, `data` will `undefined`
  - in case it failed to retrieve the session, `data` will be `null`
  - in case of success, `data` will be [`Session`](https://github.com/nextauthjs/next-auth/blob/8ff4b260143458c5d8a16b80b11d1b93baa0690f/types/index.d.ts#L437-L444).
- **`status`**: enum mapping to three possible session states: `"loading" | "authenticated" | "unauthenticated"`

### Require session

Due to the way how Next.js handles `getServerSideProps` and `getInitialProps`, every protected page load has to make a server-side request to check if the session is valid and then generate the requested page (SSR). This increases server load, and if you are good with making the requests from the client, there is an alternative. You can use `useSession` in a way that makes sure you always have a valid session. If after the initial loading state there was no session found, you can define the appropriate action to respond.

The default behavior is to redirect the user to the sign-in page, from where - after a successful login - they will be sent back to the page they started on. You can also define an `onFail()` callback, if you would like to do something else:

#### Example

```jsx title="pages/protected.jsx"
import { useSession } from "next-auth/react"

export default function Admin() {
  const { status } = useSession({
    required: true,
    onUnauthenticated() {
      // The user is not authenticated, handle it here.
    },
  })

  if (status === "loading") {
    return "Loading or not authenticated..."
  }

  return "User is logged in"
}
```

### Custom Client Session Handling

Due to the way Next.js handles `getServerSideProps` / `getInitialProps`, every protected page load has to make a server-side request to check if the session is valid and then generate the requested page. This alternative solution allows for showing a loading state on the initial check and every page transition afterward will be client-side, without having to check with the server and regenerate pages.

```js title="pages/admin.jsx"
export default function AdminDashboard() {
  const { data: session } = useSession()
  // session is always non-null inside this page, all the way down the React tree.
  return "Some super secret dashboard"
}

AdminDashboard.auth = true
```

```jsx title="pages/_app.jsx"
export default function App({
  Component,
  pageProps: { session, ...pageProps },
}) {
  return (
    <SessionProvider session={session}>
      {Component.auth ? (
        <Auth>
          <Component {...pageProps} />
        </Auth>
      ) : (
        <Component {...pageProps} />
      )}
    </SessionProvider>
  )
}

function Auth({ children }) {
  // if `{ required: true }` is supplied, `status` can only be "loading" or "authenticated"
  const { status } = useSession({ required: true })

  if (status === "loading") {
    return <div>Loading...</div>
  }

  return children
}
```

It can be easily extended/modified to support something like an options object for role based authentication on pages. An example:

```jsx title="pages/admin.jsx"
AdminDashboard.auth = {
  role: "admin",
  loading: <AdminLoadingSkeleton />,
  unauthorized: "/login-with-different-user", // redirect to this url
}
```

Because of how `_app` is written, it won't unnecessarily contact the `/api/auth/session` endpoint for pages that do not require authentication.

More information can be found in the following [GitHub Issue](https://github.com/nextauthjs/next-auth/issues/1210).

### NextAuth.js + React Query

You can create your own session management solution using data fetching libraries like [React Query](https://tanstack.com/query/v4/docs/adapters/react-query) or [SWR](https://swr.vercel.app). You can use the [original implementation of `@next-auth/react-query`](https://github.com/nextauthjs/react-query) and look at the [`next-auth/react` source code](https://github.com/nextauthjs/next-auth/blob/main/packages/next-auth/src/react/index.tsx) as a starting point.

---

## getSession()

- Client Side: **Yes**
- Server Side: **No** (See: [`unstable_getServerSession()`](/configuration/nextjs#unstable_getserversession)

NextAuth.js provides a `getSession()` helper which should be called **client side only** to return the current active session.

On the server side, **this is still available to use**, however, we recommend using `unstable_getServerSession` going forward. The idea behind this is to avoid an additional unnecessary `fetch` call on the server side. For more information, please check out [this issue](https://github.com/nextauthjs/next-auth/issues/1535).

:::note
The `unstable_getServerSession` only has the prefix `unstable_` at the moment, because the API may change in the future. There are no known bugs at the moment and it is safe to use. If you discover any issues, please do report them as a [GitHub Issue](https://github.com/nextauthjs/next-auth/issues) and we will patch them as soon as possible.
:::

This helper is helpful in case you want to read the session outside of the context of React.

When called, `getSession()` will send a request to `/api/auth/session` and returns a promise with a [session object](https://github.com/nextauthjs/next-auth/blob/main/packages/next-auth/src/core/types.ts#L407-L425), or `null` if no session exists.

```js
async function myFunction() {
  const session = await getSession()
  /* ... */
}
```

Read the tutorial [securing pages and API routes](/tutorials/securing-pages-and-api-routes) to know how to fetch the session in server side calls using `unstable_getServerSession()`.

---
## updateSession()

- Client Side: **Yes**
- Server Side: No

The `updateSession()` method can be used client-side to update the SessionProvider's state. This is especially useful, when the token or session has to be updated without a full page reload.

#### Example

```js
import { updateSession } from "next-auth/react"
const updateUser = async () => {
  // Update the user, e.g. via a POST request
  // Then update the SessionProvider's state
  await updateSession()
}
```

---
## getCsrfToken()

- Client Side: **Yes**
- Server Side: **Yes**

The `getCsrfToken()` method returns the current Cross Site Request Forgery Token (CSRF Token) required to make POST requests (e.g. for signing in and signing out).

You likely only need to use this if you are not using the built-in `signIn()` and `signOut()` methods.

#### Client Side Example

```js
async function myFunction() {
  const csrfToken = await getCsrfToken()
  /* ... */
}
```

#### Server Side Example

```js
import { getCsrfToken } from "next-auth/react"

export default async (req, res) => {
  const csrfToken = await getCsrfToken({ req })
  /* ... */
  res.end()
}
```

---

## getProviders()

- Client Side: **Yes**
- Server Side: **Yes**

The `getProviders()` method returns the list of providers currently configured for sign in.

It calls `/api/auth/providers` and returns a list of the currently configured authentication providers.

It can be useful if you are creating a dynamic custom sign in page.

---

#### API Route

```jsx title="pages/api/example.js"
import { getProviders } from "next-auth/react"

export default async (req, res) => {
  const providers = await getProviders()
  console.log("Providers", providers)
  res.end()
}
```

:::note
Unlike and `getCsrfToken()`, when calling `getProviders()` server side, you don't need to pass anything, just as calling it client side.
:::

---

## signIn()

- Client Side: **Yes**
- Server Side: No

Using the `signIn()` method ensures the user ends back on the page they started on after completing a sign in flow. It will also handle CSRF Tokens for you automatically when signing in with email.

The `signIn()` method can be called from the client in different ways, as shown below.

### Redirects to sign in page when clicked

```js
import { signIn } from "next-auth/react"

export default () => <button onClick={() => signIn()}>Sign in</button>
```

### Starts OAuth sign-in flow when clicked

By default, when calling the `signIn()` method with no arguments, you will be redirected to the NextAuth.js sign-in page. If you want to skip that and get redirected to your provider's page immediately, call the `signIn()` method with the provider's `id`.

For example to sign in with Google:

```js
import { signIn } from "next-auth/react"

export default () => (
  <button onClick={() => signIn("google")}>Sign in with Google</button>
)
```

### Starts Email sign-in flow when clicked

When using it with the email flow, pass the target `email` as an option.

```js
import { signIn } from "next-auth/react"

export default ({ email }) => (
  <button onClick={() => signIn("email", { email })}>Sign in with Email</button>
)
```

### Specifying a `callbackUrl`

The `callbackUrl` specifies to which URL the user will be redirected after signing in. It defaults to the current URL of a user.

You can specify a different `callbackUrl` by specifying it as the second argument of `signIn()`. This works for all providers.

e.g.

- `signIn(undefined, { callbackUrl: '/foo' })`
- `signIn('google', { callbackUrl: 'http://localhost:3000/bar' })`
- `signIn('email', { email, callbackUrl: 'http://localhost:3000/foo' })`

The URL must be considered valid by the [redirect callback handler](/configuration/callbacks#redirect-callback). By default it requires the URL to be an absolute URL at the same host name, or a relative url starting with a slash. If it does not match it will redirect to the homepage. You can define your own [redirect callback](/configuration/callbacks#redirect-callback) to allow other URLs.

### Using the `redirect: false` option

:::note
The redirect option is only available for `credentials` and `email` providers.
:::

In some cases, you might want to deal with the sign in response on the same page and disable the default redirection. For example, if an error occurs (like wrong credentials given by the user), you might want to handle the error on the same page. For that, you can pass `redirect: false` in the second parameter object.

e.g.

- `signIn('credentials', { redirect: false, password: 'password' })`
- `signIn('email', { redirect: false, email: 'bill@fillmurray.com' })`

`signIn` will then return a Promise, that resolves to the following:

```ts
{
  /**
   * Will be different error codes,
   * depending on the type of error.
   */
  error: string | undefined
  /**
   * HTTP status code,
   * hints the kind of error that happened.
   */
  status: number
  /**
   * `true` if the signin was successful
   */
  ok: boolean
  /**
   * `null` if there was an error,
   * otherwise the url the user
   * should have been redirected to.
   */
  url: string | null
}
```

### Additional parameters

It is also possible to pass additional parameters to the `/authorize` endpoint through the third argument of `signIn()`.

See the [Authorization Request OIDC spec](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest) for some ideas. (These are not the only possible ones, all parameters will be forwarded)

e.g.

- `signIn("identity-server4", null, { prompt: "login" })` _always ask the user to re-authenticate_
- `signIn("auth0", null, { login_hint: "info@example.com" })` _hints the e-mail address to the provider_

:::note
You can also set these parameters through [`provider.authorizationParams`](/configuration/providers/oauth#options).
:::

:::note
The following parameters are always overridden server-side: `redirect_uri`, `state`
:::

---

## signOut()

- Client Side: **Yes**
- Server Side: No

In order to logout, use the `signOut()` method to ensure the user ends back on the page they started on after completing the sign out flow. It also handles CSRF tokens for you automatically.

It reloads the page in the browser when complete.

```js
import { signOut } from "next-auth/react"

export default () => <button onClick={() => signOut()}>Sign out</button>
```

### Specifying a `callbackUrl`

As with the `signIn()` function, you can specify a `callbackUrl` parameter by passing it as an option.

e.g. `signOut({ callbackUrl: 'http://localhost:3000/foo' })`

The URL must be considered valid by the [redirect callback handler](/configuration/callbacks#redirect-callback). By default, it requires the URL to be an absolute URL at the same host name, or you can also supply a relative URL starting with a slash. If it does not match it will redirect to the homepage. You can define your own [redirect callback](/configuration/callbacks#redirect-callback) to allow other URLs.

### Using the `redirect: false` option

If you pass `redirect: false` to `signOut`, the page will not reload. The session will be deleted, and the `useSession` hook is notified, so any indication about the user will be shown as logged out automatically. It can give a very nice experience for the user.

:::tip
If you need to redirect to another page but you want to avoid a page reload, you can try:
`const data = await signOut({redirect: false, callbackUrl: "/foo"})`
where `data.url` is the validated URL you can redirect the user to without any flicker by using Next.js's `useRouter().push(data.url)`
:::

---

## SessionProvider

Using the supplied `<SessionProvider>` allows instances of `useSession()` to share the session object across components, by using [React Context](https://reactjs.org/docs/context.html) under the hood. It also takes care of keeping the session updated and synced between tabs/windows.

```jsx title="pages/_app.js"
import { SessionProvider } from "next-auth/react"

export default function App({
  Component,
  pageProps: { session, ...pageProps },
}) {
  return (
    <SessionProvider session={session}>
      <Component {...pageProps} />
    </SessionProvider>
  )
}
```

If you pass the `session` page prop to the `<SessionProvider>` – as in the example above – you can avoid checking the session twice on pages that support both server and client side rendering.

This only works on pages where you provide the correct `pageProps`, however. This is normally done in `getInitialProps` or `getServerSideProps` on an individual page basis like so:

```js title="pages/index.js"
import { unstable_getServerSession } from "next-auth/next"
import { authOptions } from './api/auth/[...nextauth]'

...

export async function getServerSideProps({ req, res }) {
  return {
    props: {
      session: await unstable_getServerSession(req, res, authOptions)
    }
  }
}
```

If every one of your pages needs to be protected, you can do this in `getInitialProps` in `_app`, otherwise you can do it on a page-by-page basis. Alternatively, you can do per page authentication checks client side, instead of having each authentication check be blocking (SSR) by using the method described below in [alternative client session handling](#custom-client-session-handling).

### Options

The session state is automatically synchronized across all open tabs/windows and they are all updated whenever they gain or lose focus or the state changes (e.g. a user signs in or out) when `refetchOnWindowFocus` is `true`.

If you have session expiry times of 30 days (the default) or more then you probably don't need to change any of the default options in the Provider. If you need to, you can trigger an update of the session object across all tabs/windows by calling [`getSession()`](/getting-started/client#getsession) from a client side function.

However, if you need to customize the session behavior and/or are using short session expiry times, you can pass options to the provider to customize the behavior of the `useSession()` hook.

```jsx title="pages/_app.js"
import { SessionProvider } from "next-auth/react"

export default function App({
  Component,
  pageProps: { session, ...pageProps },
}) {
  return (
    <SessionProvider
      session={session}
      // In case you use a custom path and your app lives at "/cool-app" rather than at the root "/"
      basePath="cool-app"
      // Re-fetch session every 5 minutes
      refetchInterval={5 * 60}
      // Re-fetches session when window is focused
      refetchOnWindowFocus={true}
    >
      <Component {...pageProps} />
    </SessionProvider>
  )
}
```

:::note
**These options have no effect on clients that are not signed in.**

Every tab/window maintains its own copy of the local session state; the session is not stored in shared storage like localStorage or sessionStorage. Any update in one tab/window triggers a message to other tabs/windows to update their own session state.

Using low values for `refetchInterval` will increase network traffic and load on authenticated clients and may impact hosting costs and performance.
:::

#### Base path

If you are using a custom base path, and your application entry point is not at the root of the domain "/" but something else, for example "/my-app/" you can use the `basePath` prop to make NextAuth.js aware of that so that all redirects and session handling work as expected.

#### Refetch interval

The `refetchInterval` option can be used to contact the server to avoid a session expiring.

When `refetchInterval` is set to `0` (the default) there will be no session polling.

If set to any value other than zero, it specifies in seconds how often the client should contact the server to update the session state. If the session state has expired when it is triggered, all open tabs/windows will be updated to reflect this.

The value for `refetchInterval` should always be lower than the value of the session `maxAge` [session option](/configuration/options#session).

#### Refetch On Window Focus

The `refetchOnWindowFocus` option can be used to control whether it automatically updates the session state when you switch a focus on tabs/windows.

When `refetchOnWindowFocus` is set to `true` (the default) tabs/windows will be updated and initialize the components' state when they gain or lose focus.

However, if it was set to `false`, it stops re-fetching the session and the components will stay as it is.

:::note
See [**the Next.js documentation**](https://nextjs.org/docs/advanced-features/custom-app) for more information on **\_app.js** in Next.js applications.
:::

### Custom base path
When your Next.js application uses a custom base path, set the `NEXTAUTH_URL` environment variable to the route to the API endpoint in full - as in the example below and as explained [here](/configuration/options#nextauth_url).

Also, make sure to pass the `basePath` page prop to the `<SessionProvider>` – as in the example below – so your custom base path is fully configured and used by NextAuth.js.

#### Example
In this example, the custom base path used is `/custom-route`.

```
NEXTAUTH_URL=https://example.com/custom-route/api/auth
```

```jsx title="pages/_app.js"
import { SessionProvider } from "next-auth/react"
export default function App({
  Component,
  pageProps: { session, ...pageProps },
}) {
  return (
    <SessionProvider session={session} basePath="/custom-route/api/auth">
      <Component {...pageProps} />
    </SessionProvider>
  )
}
```
