---
author: Alexis Diaz
pubDatetime: 2023-02-15T12:00:00
title: Error handling with Suspense and SWR
postSlug: error-handling-with-suspense-and-swr
featured: true
draft: false
tags:
  - react
ogImage: ""
description: How Suspense allows you to use Error Boundaries to handle different ways of presenting errors in your UI.
---

# SWR and Suspense for error handling

> This is a blog post based on this video from Sam Selikoff https://www.youtube.com/watch?v=h_vVsPwvcsg&t=313s&ab_channel=SamSelikoff, go watch it and support the channel!

I like to have things at hand for reading in the same context so I'm planning on converting videos I like to blog entries that I can consult later.

The gist of the video is to show how Suspense allows you to use Error Boundaries for network requests by turning asynchronous exceptions into render-time errors.

Let's imagine we have three card components that query three different endpoints with user account information. There are a few possibilities we need to take into account to render them and handle what happens in case of an error.

- One could be that any endpoint that fails to fetch data we show an error on the respective card,
- The other would be in case any of them fails we show an error and don't display any card.

This last one is a bit tricky, since two of three endpoints (or one whatever) could finish earlier than the one it throws an error, so we would display relevant information but then show an error that replaces the entire UI, which looks a bit weird.

[insert image swr-suspense-error-handling-1.png]

Let's go over how we would handle the first case,

- For the example is using SWR but I think it could be ReactQuery also

```jsx
export default function App(props) {
  return (
    <SWRConfig
      value={{
        fetcher: (...args) =>
          fetch(...args).then(res => {
            if (res.ok) {
              return res.json();
            } else {
              throw new Error("Fetch failed");
            }
          }),
      }}
    >
      <AppInner {...props} />
    </SWRConfig>
  );
}

function AppInner({ Component, pageProps }) {
  return (
    <div className="xs:px-20 md:pt-28 bg-slate-100 flex min-h-screen w-full justify-center pt-8 antialiased sm:px-10">
      <div className="w-full max-w-sm px-8">
        <Component {...pageProps} />
      </div>
    </div>
  );
}
```

- This is our mock API

```js
import faker from "faker";
import { createServer, Response } from "miragejs";

faker.seed(5);

export function makeServer({ environment = "test" } = {}) {
  let server = createServer({
    environment,

    timing: 750,

    routes() {
      this.namespace = "api";

      this.get(
        "/checking",
        () => {
          // Force an error
          // return new Response(500);
          return {
            stat: "$8,017",
            change: "$678",
            changeType: "increase",
          };
        },
        { timing: 500 }
      );

      this.get(
        "/savings",
        () => {
          return {
            stat: "$24,581",
            change: "$1,167",
            changeType: "decrease",
          };
        },
        { timing: 1500 }
      );

      this.get(
        "/credit",
        () => {
          return {
            stat: "$4,181",
            change: "$412",
            changeType: "increase",
          };
        },
        { timing: 750 }
      );

      this.namespace = "";
      this.passthrough();
    },
  });

  // Don't log passthrough
  server.pretender.passthroughRequest = () => {};

  server.logging = false;

  return server;
}

let isClient = typeof window !== "undefined";
if (isClient && !window.server) {
  window.server = makeServer({ environment: "development" });
}
```

The component that renders the information

```jsx
import { ArrowSmDownIcon, ArrowSmUpIcon } from "@heroicons/react/solid";
import useSWR from "swr";

export default function Stat({ Icon, label, endpoint }) {
  let { data } = useSWR(endpoint);

  return (
    <div className="flex w-full items-center">
      <div className="shrink-0">
        <Icon className="text-slate-300 h-10 w-10" />
      </div>
      <div className="pl-5">
        <p className="text-slate-500 truncate text-sm font-medium">{label}</p>
        <div className="flex items-baseline">
          <p className="text-slate-900 text-2xl font-semibold">{data.stat}</p>
          <p
            className={`ml-2 flex items-baseline text-sm font-semibold ${
              data.changeType === "increase" ? "text-green-600" : "text-red-600"
            }`}
          >
            {data.changeType === "increase" ? (
              <ArrowSmUpIcon className="text-green-500 h-5 w-5 shrink-0 self-center" />
            ) : (
              <ArrowSmDownIcon className="text-red-500 h-5 w-5 shrink-0 self-center" />
            )}
            {data.change}
          </p>
        </div>
      </div>
    </div>
  );
}
```

And this is the component that wraps all of them, each `Card` needs to be wrapped into `ErrorBoundary` (3rd party library, there are other options) and `Suspense` to handle the loading state.

With this we handle the first scenario, any endpoint that fails to fetch data we show an error only on the respective card and the others will show the information properly, or an error if they also fail of course.

```jsx
import { Suspense } from "react";
import { ErrorBoundary } from "react-error-boundary";
import Card from "../components/card";
import Error from "../components/error";
import Stat from "../components/stat";
import * as Icons from "@heroicons/react/outline";
import Spinner from "../components/spinner";

export default function Home() {
  return (
    <>
      <h3 className="text-slate-900 mt-2 mb-5 text-lg font-medium leading-6">
        Your accounts
      </h3>

      <div className="grid grid-cols-1 gap-5">
        <ErrorBoundary fallback={<Error>Could not fetch data.</Error>}>
          <Suspense fallback={<Spinner />}>
            <Card>
              <Stat
                label="Checking"
                endpoint="/api/checking"
                Icon={Icons.CashIcon}
              />
            </Card>
          </Suspense>
        </ErrorBoundary>

        <ErrorBoundary fallback={<Error>Could not fetch data.</Error>}>
          <Suspense fallback={<Spinner />}>
            <Card>
              <Stat
                label="Savings"
                endpoint="/api/savings"
                Icon={Icons.CurrencyDollarIcon}
              />
            </Card>
          </Suspense>
        </ErrorBoundary>

        <ErrorBoundary fallback={<Error>Could not fetch data.</Error>}>
          <Suspense fallback={<Spinner />}>
            <Card>
              <Stat
                label="Credit Card"
                endpoint="/api/credit"
                Icon={Icons.CreditCardIcon}
              />
            </Card>
          </Suspense>
        </ErrorBoundary>
      </div>
    </>
  );
}
```

And to handle the other scenario, in which any of them fails we show an error and don't display any card information, waiting for all of them to resolve or throw an error. In this case we wrap all the cards with `ErrorBoundary` and `Suspense`.

```jsx
import { Suspense } from "react";
import { ErrorBoundary } from "react-error-boundary";
import Card from "../components/card";
import Error from "../components/error";
import Stat from "../components/stat";
import * as Icons from "@heroicons/react/outline";
import Spinner from "../components/spinner";

export default function Home() {
  return (
    <>
      <h3 className="text-slate-900 mt-2 mb-5 text-lg font-medium leading-6">
        Your accounts
      </h3>

      <ErrorBoundary fallback={<Error>Could not fetch data.</Error>}>
        <Suspense fallback={<Spinner />}>
          <div className="grid grid-cols-1 gap-5">
            <Card>
              <Stat
                label="Checking"
                endpoint="/api/checking"
                Icon={Icons.CashIcon}
              />
            </Card>

            <Card>
              <Stat
                label="Savings"
                endpoint="/api/savings"
                Icon={Icons.CurrencyDollarIcon}
              />
            </Card>

            <Card>
              <Stat
                label="Credit Card"
                endpoint="/api/credit"
                Icon={Icons.CreditCardIcon}
              />
            </Card>
          </div>
        </Suspense>
      </ErrorBoundary>
    </>
  );
}
```
