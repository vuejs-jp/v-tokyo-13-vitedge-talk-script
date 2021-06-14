# Introduction

Hi everyone, my name is Fran Dios, I'm a fullstack developer who loves JavaScript in general, and Vue.js in particular.
Today, we are going to learn about Edge Computing in general and how to do Server Side Rendering using Vue.js at the edge. For that, we are going to see three different technologies:

- Cloudflare Workers, a disruptive serverless platform to deploy JavaScript code to the edge.
- Vite, which is a new frontend tooling to develop and build web apps.
- And then, how to combine them in order to run our web apps at the edge using a framework that I'm currently working on called Vitedge.

I hope you find this topic as interesting as I do and enjoy the talk. Let's get started.

# The Lie of Serverless

First of all, let me put you in context. I bet that you have heard about the word "serverless" before, also known as "cloud functions" or "lambda". When I first heard about this a few years ago, I imagined that it would allow my APIs to run world-wide with low latency and high scalability. In reality, this "serverless" was only half of what I expected.

It's true that it is scalable and has a metered billing. However, in general, it only runs in 1 place in the world (normally in the US).

On top of that, these cloud functions suffer from something called "cold start", which basically means that our function might be removed from the server's memory a while after it runs, and then, in the next request, the server will need to load the function in memory again. This process can take some time, as we can see in this graph (see image). AWS: up to 0.5 seconds; GCP up to 1.5 seconds; Azure up to 4 seconds.

Apart from that, cloud functions are stateless, meaning that its memory and data cannot be shared, generally, among different requests. This is normally good because it helps with memory leaks and other errors. However, it might make some tasks more complicated than in a "stateful" environment, such as authentication and saving user sessions. You would need a separate service like Redis to help with this.

# Going truly serverless: Cloudflare Workers (1)

In contrast with the "traditional" serverless enviornment, the company Cloudflare released a while ago a new disruptive technology called Cloudflare Workers. Workers are all what I thought that the original "serverless" would be:

It enables us to deploy our JavaScript code once, and run it in more than 200 locations all over the world. There are nodes even in Africa and China.

Workers make our code and APIs in general run very fast for two main reasons:

- The code runs very near our users or customers. If I'm in Tokyo, a node in Narita will handle my request. If I'm in Fukuoka, my request will be likley handled by the node in Osaka instead, which is closer. This means the network latency is reduced by several oders of magnitude since the information does not need to travel to America and come back.
- The second reason is that, unlike traditional serverless platforms, Workers have no cold start. It literally takes 0 milliseconds to start running your code.

The code is very scalable because each worker can be spawned million of times, again with 0 milliseconds cold start.

Workers also provide location utilities such as user's country, city or even postal code, which unlocks many possibilities. You just need to inspect the request object in your API to access these properties.

# Going truly serverless: Cloudflare Workers (2)

You might think I'm done selling you workers but no, there are more goodies. Workers also provide:

- A flexible edge cache. This means once our API handles a request, we can optionally choose to save the result so the next request directly gets a copy, which makes it even faster.

- Remember the stateless issue I mentioned earlier? Well, even though workers themselves are stateless, they have access to a built-in global Key-Value store where we can save user session, static files or anything we need.

- Recently they added support for WebSockets. Generally, a stateless function shouldn't be able to store an open websocket connection -- because the connection would be closed after the code runs. However, thanks to a side service called Durable Objects, we can now save open WebSocket connections globally. This means we can build, for example, real time videogames or collaborative tools like Google Docs. And it all still runs at the edge, near the end user.

- Of course, running at Cloudflare, we also get access to all the other services they provide because Workers are well integrated with most of them. For example, we have built-in DNS, analytics, cron jobs and security. If we have a DDOS attack we just need to press a button and Cloudflare will mitigate it by applying rate limits and whatnot.

- Last but not least, workers are generally cheaper than traditional serverless platforms.

Now, this is a lot of information but, what can we actually build with Workers?

# Workers Examples

These are some of the examples that workers are commonly used for.

- A/B testing. We can return different information based on user location or any other thing.
- Anlytics. To some degree, we can have good analytics directly from the Worker without pushing SDKs to the browser.
- Something I had to do recently: Show a banner in a Japanese online shop only to overseas customers. Users within Japan get an empty response whereas overseas users get the banner.

# Downsides of Workers

So far we have seen a lot of benefits to using Cloudflare Workers. However, there are also some downsides:

- Workers tend to follow Web APIs instead fo Node.js APIs. This means that there are a lot of NPM packages for backend that are not supported in the Worker environment, such as Stripe SDK or Auth0's JWT verification utilities. We need to use raw APIs or find utilities that rely on Web Standards instead of Node.js APIs.
- Workers run in a very constrained environment. This means there are runtime limits such as maximum memory, CPU time, or the number of subrequests we can make. Not every application can run in this environment and we need to keep that in mind when building our apps.
- Also, the developer experience is not that great yet compared to other providers, which might make debugging a bit harder, but they are improving.
- The cumminity is still young so you might need to spend more time to learn how to do certain things.

Now that we have seen what actually "deploying to the edge" means, by using Cloudflare Workers, let's have a look at Vite.

# Vite

You have probably heard of Vite. It is a new frontend tooling developed by the creator of Vue.js, Evan You.
You can think of Webpack and Webpack-dev-server combined, but infinitely faster and easier to configure.

Thanks to this, the developer experience when you create web apps with Vite is really good. Its default configuration values work for most applications and, if you need to modify the configuration, is still really simple. For example, if you need to use SASS or SCSS, just install their compilers and Vite will pick them automatically.

Vite's development server starts almost instantly, and the same happens with its Hot Module Replacement, HMR. You save a file and you get the update right away in the browser.

Apart from that, its community is flourishing. There are so many good developers creating plugins for everything you can imagine. Filesystem routing, components auto import, i18n precompilation, easy access to icons, and so on. You just need to install the pluign and import it in 1 line of code.

Vite is framework agnostic but, of course, since its made by Evan You, you can imagine it works flawlessly with Vue.js.

And apart from that, one of the things I like the most is that Vite is really easy to extend. It exports most of its internal functions so we can easily create tools on top of it for handling WebSockets or Server Side Rendering.

And this is precisely what I've been working on during the last months. Running Vite SSR tooling at the edge, in Cloudflare Workers.

# Vitedge: SSR at the Edge (1)

And that's what I've called Vitedge, which is just a Vite application that, when you build it and deploy it to a Cloudflare Worker, it can render the HTML for your page components at the edge, very near the end user. Even though that sounds complex, it is still as easy as developing a Single Page Application.

Therefore, instead of uploading and serving a static `index.html` from the CDN, its the CDN node itself the one that renders the `index.html` file and caches at the edge it for the subsequent requests. So the next request will just get the file as if it was a static file.

# Vitedge: SSR at the Edge (2)

You can probably imagine some of the benefit sof this:

- Better loading times and SEO without slow builds and supporting dynamic content.
- Great performance.
- Configurable cache. This means that, if we know that our data (from a database or a content management system) has changed for a specific route, we can simply evict that route from cache and keep the others untouched. So the next request for that route will use fresh data.
- Vitedge also provids a way to create API endpoints based on filesystem routes, which is really simple.
- And of course, we have all the good DX that Vite brings.

I also want to mention that, even though SSR can be good for many applications, it might not be a good fit for your specific project. SPA and static site generators can still be good choices depending on the project.

OK, so let's see how we can install and use Vitedge in a Vite application.

# Vitedge: SSR at the Edge (3)

If you are not familiar with Vite yet, a normal application looks like this. We have our `vite.config` file and a normal "source" folder for any Vue.js application, with page components, routes, the root component (App) and the main entry file for Vue apps.

The way we install Vitedge is by simply adding its plugin to the Vite config, and modifying the entry point of the Vue application like this. Vitedge will call `createApp` or `createSsrApp` depending on the environment. In this main hook we can setup any Vue.js modules such as i18n, Vuex, Pinia and so on.

By default, just like in a Vue SPA, in Vitedge we also have a single entry point, instead of having 1 for the client and 1 for the server. This makes it easier to get started with SSR, although using 2 separate entry points is also supported in case you need it.

# Vitedge: SSR at the Edge (4)

Now, let's see how we can create a Page component that gets data from our API.
In Vitedge, we place our API in a folder called `functions` next to the `source` folder for the frontend.

Here we have a simple Page component to show a Blog Post, and we put it in a route called "post" with path "/posts/:slug".
You can see the Page component expects to get the Post data from its props. And then it uses that data to write some meta tags using `vueuse/head` and it displays some of its content in the DOM.

Now, how do we get to provide this Post object to the Page component? Well, if you remember, the route for this page is called `post`. We just need to create a new file under "functions/props/" directory with the same name of that route, "functions/props/post.ts" (TS or JS). This function here will be called everytime we access this route in our browser, and it will pass its result to the Page component as props. You can see that the route expects a "slug" parameter, and this is actually provided to the props function in its arguments.

Here, we can use this "slug" parameter to get the post information from our database or our content management system using `fetch`, and the return the result.

As simple as that. This function code is not bundled in the web application so you can have here any private information such as API keys. Also, notice that we are specifying a cache value here. This means that the rendered HTML for this page component will be cached at the edge for this amount of seconds. If we notice that our CMS or DB data for this post has changed, we can make a request to Cloudflare's cache and remove this information, so the next request will use fresh data.

Notice that we also have here some files such as `.env` and sitemap. The `.env` file simply stores environment variables that are available to the API, whereas the sitemap file is just like this function here (props handler) but it will be called any time we reach `/sitemap.xml` or `/sitemap.txt` in the browser so we can return a dynamic sitemap that depends on our DB or CMS data. And of course it can also be cached. Instead of sitemap, we could have a `robots` file or even a `graphql` file where we setup a GraphQL server.

And that's all for today. If you want to know more about Vitedge, please have a look at https://vitedge.js.org