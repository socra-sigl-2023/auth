# Authentication and authorization workshop

This workshop aims student of EPITA SIGL 2023.

In this workshop, you will learn how to:
- secure your frontend behind a login page
- secure your web API 
- use [Auth0](https://auth0.com) for authentication and authorization

## Step 1: Setup your account on Auth0

- Go to [Auth0](https://auth0.com/auth/login) and login with your github account.
- Create a new EU dev tenant with the format `socra-groupXX` (e.g. `socra-group20` for group 20)
![tenant.png](images/tenant.png)

To use Auth0 service in yours app, you need credentials.

To get them, go to the `Applications` section in Auth0 dashboard.

A default `Application` is already available but we will create a new one called Arla:

- Click on `Create Application` button
- Name it `Socra` and select `Single Page Web Application`
- Click on Create

> Note: The dashboard propose you to see a Quick Start with the different technology.
> Feel free to check available Quick Start for your favourite tech.
> 
> We adapted a bit the code provided from [React Quick Start](https://auth0.com/docs/quickstart/spa/react/01-login) for today's workshop.

- You have credentials available in the `Settings` tab of your `Socra` `Application`.
- You need to configure the Callback URLs, Logout URLs, Allowed Web Origins. Put these URLs (separate them with comma), replacing XX by your group number:
  - http://localhost:8080             (use for local dev environment)
  - http://localhost:3030
  - https://groupXX.socra-sigl.fr     (use for production environment)
  - https://api.groupXX.socra-sigl.fr

- Save changes

Before moving to code changes, let's create a test user in your Auth0 Application:
- From your Auth0 dashboard, move to user managment
![user-managment.png](images/user-managment.png)
- Click on `Create User` with credentials you'll remember and can share to your group
![add-user.png](images/add-user.png)


## Step 2: Integrate Auht0 login to your frontend

**Objective**: Allow **ONLY** authenticated users to access Socrate.

### Install the Auth0 React SDK

Auth0 provide an easy to use sdk for react.

First you need to install [Auth0 React SDK](https://www.npmjs.com/package/@auth0/auth0-react), in your `frontend/` directory:

```sh
# from frontend/
nvm use 16
npm i --save @auth0/auth0-react
```

You can now import the sdk and start to configure the `Auth0Provider` component.

> Note: Under the hood, the Auth0 React SDK uses React Context to manage the authentication state of your users.
> One way to integrate Auth0 with your React app is to wrap your root component 
> with an Auth0Provider that you can import from the SDK.

In your `frontend/src/App.js`, you will add:
- `auth0` import
  ```js
  import { Auth0Provider } from "@auth0/auth0-react";
  ```
- `Auth0Provider` provided by Auth0; configured with your Auth0 Socra's application's settings

```javascript
// inside frontend/src/App.js
//...
import { Auth0Provider } from "@auth0/auth0-react";

function App() {
  const [state, dispatch] = React.useReducer(reducer, initialState);
  return (
    <Auth0Provider 
      clientId="<YOUR CLIENT ID>"
      domain="socra-groupXX.eu.auth0.com"
      redirectUri={window.location.origin}
      cacheLocation="localstorage"
    >
      <SocrateContext.Provider value={{ state, dispatch }}>
        {/* ... */}
      </SocrateContext.Provider>
    </Auth0Provider>
  );
}

```

> Note: The Auth0Provider component takes the following props:
> - `YOUR_DOMAIN` and `YOUR_CLIENT_ID`: The values of these properties correspond to the "Domain" and "Client ID" values 
> present under the "Settings" of the single-page application that you registered with Auth0.
>
> - `redirectUri`: The URL to where you'd like to redirect your users after they authenticate with Auth0.
> 
> `Auth0Provider` stores the authentication state of your users and the state of the SDK — whether Auth0 is ready to use or not.
> It also exposes helper methods to log in and log out your users, which you can access using the `useAuth0` hook.

### Add Login

The Authenticated is here to automaticly redirect the user to Auth0 when it's not logged.

- Create a file `frontend/src/Authenticated.js` with:

```jsx
import React from "react";
import { useAuth0 } from "@auth0/auth0-react";

export function Authenticated({ children }) {
  const { loginWithRedirect, user, isLoading } = useAuth0();

  React.useEffect(() => {
    async function redirect() {
      if (!user && !isLoading) {
        await loginWithRedirect();
      }
    };
    redirect();
  }, [isLoading]);
  
  return isLoading ? <span>Loading ...</span> : <>{children}</>;
};
```

The `Authenticated` React component implements this logic:
    - if user is not authenticated, you are redirecting user to login page.
    - if user is logged in, you are rendering child components.

Now, use the `Authenticated` React component you've used to secure all the `App`'s content behind a login page.
- In `frontend/src/App.js`, wrap `SocrateContext` provider with the `Authenticated` react component:
```jsx
// ...
import { Authenticated } from "./Authenticated";

function App() {
  // ...
  return (
    <Auth0Provider
      // ...
    >
      <Authenticated>
        <SocrateContext.Provider value={{ state, dispatch }}>
          // ...
        </SocrateContext.Provider>
      </Authenticated>
    </Auth0Provider>
  );
}

export default App;
```

Congratulation, you need successfully added login to Socrate!

Try it out, run your frontend and browse http://localhost:8080

### Add Logout to your Application

The `useAuth0` react hook is providing a `logout` method for the user to terminate her/his session.

- Create a new file `frontend/src/Logout.js` with:

```jsx
import React from "react";
import { useAuth0 } from "@auth0/auth0-react";

function Logout() {
  const { logout } = useAuth0();
  return (
    <a onClick={() => {
        logout({ returnTo: document.location.origin });
      }}
    >
      Déconnexion
    </a>
  );
}

export default Logout;
```
- Integrate `Logout` React component to the `Navigation` component. Adapt `frontend/src/Navigation.js`:
```js
// ...
import Logout from "./Logout";
// ...
function Navigation() {
  const menuItems = [
    { name: "Catalogue", to: "/" },
    { name: "Commandes", to: NavigationLinks.orders },
    { name: <ShoppingCartIcon />, to: NavigationLinks.basket },
  ];

  return (
    <div className="pure-menu pure-menu-horizontal">
      <NavLink to="/" className="pure-menu-heading">
        SOCRATE
      </NavLink>
      <nav className="pure-menu-list">
        {menuItems.map((menuItem, idx) => (
          <MenuItem key={idx} menuItem={menuItem} />
        ))}
      </nav>
      <div className="menu-right">
        <Logout />
      </div>
    </div>
  );
};
```
- And replace `frontend/src/Navigation.css` with:
```css
/*
   * -- MENU STYLES --
   * Make the menu have a very faint box-shadow.
   */
.pure-menu {
  box-shadow: 0 1px 1px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: row;
}

.pure-menu-link {
  padding: 0.5em 0.7em;
}

.menu-right {
  flex-grow: 1;
  align-self: center;
  text-align: right;
  margin-right: 5px;
  cursor: pointer;
}
```

You should be all set! 

Start your frontend code
```sh
# from frontend/
npm start
```

And you should have logout working correctly in the navigation on [locahost:8080](http://localhost:8080)

## Step 3: Secure your API

**Objective**: Only user who are **logged in** can consume your web API.

### Create an API in Auth0

You need to create credentials for your web API in the same Auth0 tenant:

- go to your Dashboard > API > Create new API
![api-menu.png](images/api-menu.png)
- Set the following fields (adapt for your group name):
  - Name: socra-groupXX-api
  - Identifier: https://api.groupXX.socra-sigl.fr
  - Leave default signing algorithm: RS256

You should see quick-start and your API created with an identifier; something like:
![create-api.png](images/create-api.png)

From your `backend/` code, adapt with the code Auth0 provides in its API QuickStart > NodeJS section:

- install dependencies

```sh
# from backend/
nvm use v16
npm install --save express-jwt jwks-rsa
```

- Adapt the following code `jwtCheck` variable with your own group's context. Verify your credentials by looking in your Auth0 API QuickStart NodeJS section (like screenshot above):

```js
// src/server.js
//...
const jwt = require('express-jwt');
const jwks = require('jwks-rsa');
//...

const jwtCheck = jwt({
    secret: jwks.expressJwtSecret({
        cache: true,
        rateLimit: true,
        jwksRequestsPerMinute: 5,
        jwksUri: 'https://socra-groupXX.eu.auth0.com/.well-known/jwks.json'
    }),
    audience: 'https://api.groupXX.socra-sigl.fr',
    issuer: 'https://socra-groupXX.eu.auth0.com/',
    algorithms: ['RS256']
});

app.use(jwtCheck);

// ...
```

That's it! You've secured your API. `jwtCheck` is a middleware that will check if the `Bearer` token is valid on **all incomming requests to any routes**.

You can try out to curl your API again, you should get an `Unauthorized` message:
- 
```sh
# make sure to run your api from a terminal session;
# from backend/ 
# nvm use v16
# node src/server.js

curl 'http://localhost:3030/v1/discounts'
# You should get an HTML containing:
# UnauthorizedError: No authorization token was found
```

And if you wanna verify if giving a fake token would work, you can try (and make sure it fails!):

```sh
curl -H "Authorization: Bearer AFakeT0kenThatMeansNothingButIAmStillTrying" 'http://localhost:3030/v1/discounts'
# You should get an HTML containing:
# UnauthorizedError: jwt malformed
```

The only way to get a correct token is if you are authenticated thru the login page of frontend.

## Step 4: Call your Secured API

**Objective**: Make authenticated calls to your web API, from your frontend code.

You may have noticed the usage of [fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) to consume your web API inside `frontend/src/utils.js`.

In this step, you will:
- Setup your audience
- Adapt fetch to include a secured token in request's headers

### Adapt your frontend component in the frontend

Set the `audience` property of your `<Auth0Provider />` to your `API identifier` from your newly created API in Auth0:
![api-audience.png](images/api-audience.png)
```jsx
// inside frontend/src/App.js
// ...
<Auth0Provider
    // ...
    audience="Your API Identifier"
    cacheLocation="localstorage"
  >
```

This will make sure the token you get after authentication has access your API.

### Adapt fetch

You will use the provide method `getAccessTokenSilently` from the `useAuth0` hook provided by Auth0 SDK.

In your `frontend/src/utils.js`, add the following changes:

```js
export async function callApi(token, route) {
  const endpoint = `${SOCRATE_API_URL}${route}`;
  const response = await fetch(endpoint, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return await response.json();
}
```

How to obtain a valid token from our user session?

Make use of `useAuth0` react hook.
- Adapt `frontend/src/Discount.js` with:
```js
// ...
import { Auth0Provider } from "@auth0/auth0-react";
// ...
function Discounts() {
  const [discounts, setDiscounts] = React.useState([]);
  const { getAccessTokenSilently } = useAuth0();

  React.useEffect(function () {
    async function fetchDiscounts() {
      try {
        const token = await getAccessTokenSilently();
        const discountProducts = await callApi(token, "/v1/discounts");
        setDiscounts(discountProducts);
      } catch {
        console.log("ERROR: can't fetch discounts");
      }
    }
    fetchDiscounts();
  }, []);

  return (
    <div className="pure-g deals">
      <h1 className="pure-u-1">Les bonnes affaires</h1>
      {discounts.map((product, idx) => (
        <ProductCard key={idx} product={product} />
      ))}
    </div>
  );
}
// ...
```

`const token = await getAccessTokenSilently();` gets a valid JWT token to call socrate web API. This JWT token is valid only a limited time, that's why it needs to be dynamic.

Do the same adaption for 
- `frontend/src/Category.js` 
- and for `frontend/src/ProductCategory.js`

# Step 5: Deploy your changes

Push your changes on `main`, and wait for both `backend` and `frontend` to be deployed.

You can make sure your API is secured by querying `https://api.socra-groupXX.socra-sigl.fr/v1/discounts` without token. You should have an `unauthorized error`.

And you shouldn't be able to use https://groupXX.socra-sigl.fr/ without login first!

After login, you could see the `Authorization` header with a valid token if you inspect your browser's network:
![](images/inspect-bearer.png)
