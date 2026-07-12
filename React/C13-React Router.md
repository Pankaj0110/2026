# The two ways to Define Routers

Modern React router offers two distinct approaches to declaring your route tree.

## Approach A: The Router router (Recommended)
```js
import { createBrowserRouter, RouterProvider } from 'react-router-dom'

const router = createBrowserRouter([
  {
    path: "/",
    Component: Root,
    children: [
      { index: true, Component: Home },
      { path: "about", Component: About },
      {
        path: "auth",
        Component: AuthLayout,
        children: [
          { path: "login", Component: Login },
          { path: "register", Component: Register },
        ],
      },
      {
        path: "concerts",
        children: [
          { index: true, Component: ConcertsHome },
          { path: ":city", Component: ConcertsCity },
          { path: "trending", Component: ConcertsTrending },
        ],
      },
    ],
  },
]);

export const App = () => <RouterProvider router={router} />
```

**Complete example**

*App.js*
```js
import React from 'react';
import { Outlet } from 'react-router';

import { NavBar } from './Routes';

import './style.css';

export default function App() {
  return (
    <div>
      <NavBar />
      <Outlet />
    </div>
  );
}

```

*Index.js*
```js
import React, { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { RouterProvider } from 'react-router';

import { AppRoutes } from './Routes';

const rootElement = document.getElementById('root');
const root = createRoot(rootElement);

root.render(
  <StrictMode>
    <RouterProvider router={AppRoutes} />
  </StrictMode>
);

```

*Routes.jsx*

```js
import React from 'react';
import { Outlet, useParams, createBrowserRouter } from 'react-router';
import { Link, NavLink } from 'react-router';
import App from './App';

export const NavBar = () => {
  return (
    <div style={{ display: 'flex', flexDirection: 'row', columnGap: '20px' }}>
      <NavLink to="/">Home</NavLink>
      <NavLink to="/services">services</NavLink>
      <NavLink
        to="/product"
        className={({ isActive }) => (isActive ? 'myActive' : '')}
      >
        Product
      </NavLink>
    </div>
  );
};

const Landing = () => {
  return (
    <div>
      <h1> landing </h1>
    </div>
  );
};

const Product = () => {
  return (
    <div>
      <h1>Product</h1>
      <ul>
        <li>
          <Link to="notebook">Notebook</Link>{' '}
        </li>
        <li>
          <Link to="pen">Pen</Link>
        </li>
      </ul>
      <Outlet />
    </div>
  );
};
const ProductDetails = () => {
  const { id } = useParams();
  return (
    <div>
      <h1>ProductDetails</h1>
      {id}
    </div>
  );
};
const Services = () => {
  return (
    <div>
      <h1>Services</h1>
    </div>
  );
};

export const AppRoutes = createBrowserRouter([
  {
    path: '/',
    Component: App,
    children: [
      {
        index: true,
        Component: Landing,
      },
      {
        path: 'product',
        Component: Product,
      },
      {
        path: 'product/:id',
        Component: ProductDetails,
      },
      {
        path: 'services',
        Component: Services,
      },
    ],
  },
]);


```

**OR using jsx but use the same apis **

```js
import {
  createBrowserRouter,
  createRoutesFromElements,
  RouterProvider,
  Route,
} from "react-router-dom";

const router = createBrowserRouter(
  createRoutesFromElements(
    <>
      <Route path="/" element={<Home />} />
      <Route path="/about" element={<About />} />
    </>
  )
);

function App() {
  return <RouterProvider router={router} />;
}

```

**Before 6.4 it was `element` , after that it is `component`**

Timeline:

React Router v5 → <Switch>, component / render props
React Router v6.0 → <Routes>, element, nested routes, createRoutesFromElements
React Router v6.4 → Data Router APIs (createBrowserRouter, loaders, actions, etc.)
React Router v7 → Continues supporting these APIs with additional improvements
React Router v8 → (if using a future/release version) continues the same JSX route conversion pattern

So if you are using React Router 7.18.2, createRoutesFromElements is available. It is not a React Router 8-only feature.


## Approach B: JSX only component router (legacy/ traditional)
This is the classical, entirely render-driven layout. While familiar, it suffers from the "fetch-on-render" waterfall problem, where nested routes cannot start fetching data until their parent routes finish rendering and execution `useEffect`.

```js
import React from 'react';
import { Outlet, useParams } from 'react-router';
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

const NavBar = () => {
  return (
    <div style={{ display: 'flex', flexDirection: 'row', columnGap: '20px' }}>
      <Link to="/">Home</Link>
      <Link to="/services">services</Link>
      <Link to="/product">Product</Link>
    </div>
  );
};

const Landing = () => {
  return (
    <div>
      <h1> landing </h1>
    </div>
  );
};

const Product = () => {
  return (
    <div>
      <h1>Product</h1>
      <ul>
        <li>
          <Link to="/notebook">Notebook</Link>{' '}
        </li>
        <li>
          <Link to="/pen">Pen</Link>
        </li>
      </ul>
      <Outlet />
    </div>
  );
};
const ProductDetails = () => {
  const { id } = useParams();
  return (
    <div>
      <h1>ProductDetails</h1>
      {id}
    </div>
  );
};
const Services = () => {
  return (
    <div>
      <h1>Services</h1>
    </div>
  );
};

export const AppRoutes = () => {
  return (
    <BrowserRouter>
      <NavBar />
      <Routes>
        <Route path="/" element={<Landing />} />
        <Route path="/services" element={<Services />} />
        <Route path="/product" element={<Product />}></Route>
        <Route path=":id" element={<ProductDetails />} />
      </Routes>
    </BrowserRouter>
  );
};

```


## More about DATA routes

The objects passed to createBrowserRouter are called Route Objects.

```JS
createBrowserRouter([
  {
    path: "/",
    Component: App,
  },
]);

```

Route modules are the foundation of React Router's data features, they define

- data loading
- actions
- revalidation
- error boundaries
- & more

#### Component
The `component` property in a route object defines the componenet for particular path

#### Middleware

Route `middleware` runs sequentially before and after navigations. This gives singular place to do things like logging and authentication. The `next` function continues down the chain, and on the leaft route the `next` function executes the loaders/actions for the navigation.

```js

createBrowserRouter([
  {
    path: "/",
    middleware: [loggingMiddleware],
    loader: rootLoader,
    Component: Root,
    children: [{
      path: 'auth',
      middleware: [authMiddleware],
      loader: authLoader,
      Component: Auth,
      children: [...]
    }]
  },
]);

async function loggingMiddleware({ request }, next) {
  let url = new URL(request.url);
  console.log(`Starting navigation: ${url.pathname}${url.search}`);
  const start = performance.now();
  await next();
  const duration = performance.now() - start;
  console.log(`Navigation completed in ${duration}ms`);
}

const userContext = createContext<User>();

async function authMiddleware ({ context }) {
  const userId = getUserId();

  if (!userId) {
    throw redirect("/login");
  }

  context.set(userContext, await getUserById(userId));
};

```


#### Data loading

Data is provided to route components from route loaders:

```js
createBrowserRouter([
  {
    path: "/",
    loader: async () => {
      // return data from here
      return { records: await getSomeRecords() };
    },
    Component: MyRoute,
  },
]);

```
**Access data**

```js
import { useLoaderData } from "react-router";

function MyRoute() {
  const { records } = useLoaderData();
  return <div>{records.length}</div>;
}

```


#### Actions

Data mutations are done through Route actions defined on the action property of a route object. When the action completes, all loader data on the page is revalidated to keep your UI in sync with the data without writing any code to do it.

```js

action: async ({ request }) => {
  const data = await request.formData();

  await fetch("/api/users", {
    method: "POST",
    body: data,
  });

  return redirect("/users");
}
```



## How to Protect Routes in React router (v6)

```js

import { createContext, useContext, useState } from "react";

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(
    JSON.parse(localStorage.getItem("user"))
  );

  const login = (userData) => {
    localStorage.setItem("user", JSON.stringify(userData));
    setUser(userData);
  };

  const logout = () => {
    localStorage.removeItem("user");
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);


import { Navigate, Outlet } from "react-router-dom";
import { useAuth } from "./AuthContext";

const ProtectedRoute = () => {
  const { user } = useAuth();

  return user ? <Outlet /> : <Navigate to="/login" replace />;
};

export default ProtectedRoute;


<AuthProvider>
  <App />
</AuthProvider>

```

**Based on Roles**

```js
const ProtectedRoute = ({ allowedRoles }) => {
  const { user } = useAuth();

  if (!user) {
    return <Navigate to="/login" replace />;
  }

  if (!allowedRoles.includes(user.role)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <Outlet />;
};
```
*usage*

```js
<Route element={<ProtectedRoute allowedRoles={["admin"]} />}>
  <Route path="/admin" element={<AdminDashboard />} />
</Route>
```


### if using Router version > 6

You can use middleware (as it runs before loader & action)


```
// middleware.ts
import { redirect } from "react-router";

export async function authMiddleware({ request }) {
  const token = getToken(request);

  if (!token) {
    throw redirect("/login");
  }
}

```
