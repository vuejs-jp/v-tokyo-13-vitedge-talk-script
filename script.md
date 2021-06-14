# Introduction

Hi everyone, my name is Fran Dios, I'm a fullstack developer who loves JavaScript in general, and Vue.js in particular.
Today, we are going to learn about Edge Computing in general and how to do Server Side Rendering using Vue.js at the edge. For that, we are going to see three different technologies:

- Cloudflare Workers, a disruptive serverless platform to deploy JavaScript code to the edge.
- Vite, which is a new frontend tooling to develop and build web apps.
- And then, how to combine them in order to run our web apps at the edge using a framework that I'm currently working on called Vitedge.

I hope you find this topic as interesting as I do and enjoy the talk. Let's get started.

# サーバーレスの誤解

First of all, let me put you in context. I bet that you have heard about the word "serverless" before, also known as "cloud functions" or "lambda". When I first heard about this a few years ago, I imagined that it would allow my APIs to run world-wide with low latency and high scalability. In reality, this "serverless" was only half of what I expected.

It's true that it is scalable and has a metered billing. However, in general, it only runs in 1 place in the world (normally in the US).

On top of that, these cloud functions suffer from something called "cold start", which basically means that our function might be removed from the server's memory a while after it runs, and then, in the next request, the server will need to load the function in memory again. This process can take some time, as we can see in this graph (see image). AWS: up to 0.5 seconds; GCP up to 1.5 seconds; Azure up to 4 seconds.

Apart from that, cloud functions are stateless, meaning that its memory and data cannot be shared, generally, among different requests. This is normally good because it helps with memory leaks and other errors. However, it might make some tasks more complicated than in a "stateful" environment, such as authentication and saving user sessions. You would need a separate service like Redis to help with this.

# 真のサーバーレスとは: Cloudflare Workers (1)

In contrast with the "traditional" serverless enviornment, the company Cloudflare released a while ago a new disruptive technology called Cloudflare Workers. Workers are all what I thought that the original "serverless" would be:

It enables us to deploy our JavaScript code once, and run it in more than 200 locations all over the world. There are nodes even in Africa and China.

Workers make our code and APIs in general run very fast for two main reasons:

- The code runs very near our users or customers. If I'm in Tokyo, a node in Narita will handle my request. If I'm in Fukuoka, my request will be likley handled by the node in Osaka instead, which is closer. This means the network latency is reduced by several oders of magnitude since the information does not need to travel to America and come back.
- The second reason is that, unlike traditional serverless platforms, Workers have no cold start. It literally takes 0 milliseconds to start running your code.

The code is very scalable because each worker can be spawned million of times, again with 0 milliseconds cold start.

Workers also provide location utilities such as user's country, city or even postal code, which unlocks many possibilities. You just need to inspect the request object in your API to access these properties.

# 真のサーバーレスとは: Cloudflare Workers (2)

You might think I'm done selling you workers but no, there are more goodies. Workers also provide:

- A flexible edge cache. This means once our API handles a request, we can optionally choose to save the result so the next request directly gets a copy, which makes it even faster.

- Remember the stateless issue I mentioned earlier? Well, even though workers themselves are stateless, they have access to a built-in global Key-Value store where we can save user session, static files or anything we need.

- Recently they added support for WebSockets. Generally, a stateless function shouldn't be able to store an open websocket connection -- because the connection would be closed after the code runs. However, thanks to a side service called Durable Objects, we can now save open WebSocket connections globally. This means we can build, for example, real time videogames or collaborative tools like Google Docs. And it all still runs at the edge, near the end user.

- Of course, running at Cloudflare, we also get access to all the other services they provide because Workers are well integrated with most of them. For example, we have built-in DNS, analytics, cron jobs and security. If we have a DDOS attack we just need to press a button and Cloudflare will mitigate it by applying rate limits and whatnot.

- Last but not least, workers are generally cheaper than traditional serverless platforms.

Now, this is a lot of information but, what can we actually build with Workers?

# Workers の例

These are some of the examples that workers are commonly used for.

- A/B testing. We can return different information based on user location or any other thing.
- Anlytics. To some degree, we can have good analytics directly from the Worker without pushing SDKs to the browser.
- Something I had to do recently: Show a banner in a Japanese online shop only to overseas customers. Users within Japan get an empty response whereas overseas users get the banner.

# Workers の欠点

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

# Vitedge: エッジで動作する SSR (1)

そして、これを私はVitedgeと呼んでいますが、Viteアプリケーションを作ってそれをCouldflare Workersにデプロイすると、ページコンポーネントのHTMLをエンドユーザーに近いエッジでレンダリングすることができます。複雑に聞こえるかもしれませんが、それはシングルページアプリケーションの開発するとき同じくらい簡単です。

したがって、CDNから静的な`index.html`をアップロードして提供する代わりに、CNDノード自身が`index.html`ファイルをレンダリングし、次のリクエストのためにエッジにキャッシュします。そのため、次のリクエストでは、あたかも静的なファイルであるかのようにファイルを取得します。

# Vitedge: エッジで動作する SSR (2)

おそらく、以下のようないくつかのメリットを想像できます:

- より良いロード時間、遅いビルドなしによるSEO、動的なコンテンツをサポート
- 素晴らしいパフォーマンス
- 構成可能なキャッシュ、つまり、私達の（データベースやコンテンツ管理システムからの）データが特定のルートで変更されたことがわかっている場合は、そのルートをキャッシュから削除し、他のルートはそのままにしておくことができるのです。そのため、そのルートに対する次のリクエストは、新しいデータを使用することになります。
- Vitedgeはファイルシステムによるルートに基づいたAPIエントリポイントを作成する方法も提供します。
- そして、もちろん、Viteがもたらす良いDXもすべてあります。

SSRは多くのアプリケーションに適していいるかもしれまんが、特定のプロジェクトにおいては適していないかもしれないということも、私は伝えておきたいと思います。プロジェクトによっては、SPAと静的サイトジェネレーターは良い選択となるでしょう。

それでは、ViteアプリケーションにVitedgeをインストールして使用する方法を見てみましょう。

# Vitedge: エッジで動作する SSR (3)

もし、まだViteに慣れていない方は、通常のアプリは次のようになっています。`vite.config`ファイルとVue.jsアプリケーション向けの通常の"source"フォルダがあり、ページコンポーネント、ルート、ルートコンポーネント(App)、そしてVueアプリ用のメインエントリファイルがあります。

Vitedgeをインストールする方法は、単純にそのプラグインをViteの設定ファイルに追加し、Vueアプリケーションのエントリポイントを次のように変更します。Vitedgeは環境に応じて、`createApp`または`createSsrApp`を呼び出します。このメインフックでは、i18n、Vuex、PiniaなどあらゆるVue.jsモジュールをセットアップすることができます。

デフォルトでは、VueのSPAのように、Vitedgeではクライアントそしてサーバー向けに1つのエントリポイントを持つ代わりに1つのエントリポイントを持ちます。これによりSSRを簡単に使い始めることができます。ただ、必要に応じて2つの独立したエントリポイントを使用することもサポートされています。

# Vitedge: エッジで動作する SSR (4)

それでは、APIからデータを取得するPageコンポーネントの作成方法について説明します。
Vitedgeでは、フロントエンド向けに`source`フォルダの隣にある`functions`というフォルダにAPIを配置します。

ここでは、ブログ記事を表示するためのシンプルなPageコンポーネントを用意し、"post"というルートに"/posts/:slug"というパスで配置しています。
propsからブログポストデータを取得するために、ページコンポーネントを期待しているのが分かります。そして、それから`vueuse/head`を使っていくつかのmetaタグを書くためにそのデータを使い、DOMにいくつかのコンテンツを表示しています。

さて、このPostオブジェクトをどのようにしてページコンポーネントに提供するのでしょうか？覚えていると思いますが、このページのルートは `post`と呼ばれています。"functions/props/"ディレクトリ配下に、ルートと同じ名前の新しいファイル "functions/props/post.ts"（TSまたはJS）を作成する必要があります。この関数は、ブラウザからこのルートにアクセスするたびに呼び出され、その結果をpropsとしてページコンポーネントに渡します。ルートは "slug"パラメータを期待しており、これは実際にprops関数の引数で提供されていることがわかります。

ここでは、この"slug"パラメータを使って、データベースやコンテンツマネジメントシステムから`fetch`で投稿情報を取得し、その結果を返すことができます。

とてもシンプルですね。この関数コードはWebアプリケーションにバンドルされていないので、APIキーなどのプライベートな情報をここに持つことができます。また、ここではキャッシュ値を指定していることに注目してください。これは、このページコンポーネントのレンダリングされたHTMLが、この秒数の間、エッジにキャッシュされることを意味します。この記事のCMSやDBのデータが変更されたことに気づいたら、Cloudflareのキャッシュにリクエストしてこの情報を削除し、次のリクエストで新鮮なデータを使用できるようにします。

ここには、`.env`やsitemapといったファイルがあることに注目してください。
`.env`ファイルには、APIで利用可能な環境変数が格納されています。一方、sitemapファイルは、この関数(propsハンドラ)のようなものですが、ブラウザで`/sitemap.xml`や`/sitemap.txt`にアクセスしたときに呼び出されるので、DBやCMSのデータに依存した動的なサイトマップを返すことができます。もちろん、キャッシュにも対応しています。サイトマップの代わりに、`robots`ファイルや、GraphQLサーバーをセットアップする`graphql`ファイルを用意することもできます。

今日の発表はになります。Vitedgeについてもっと知りたい方は、https://vitedge.js.org をご覧ください。