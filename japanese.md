# Introduction

皆さん、こんにちは。私の名前は Fran Dios です。JavaScript 全般、特に Vue.js が大好きなフルスタックエンジニアです。今日は、一般的なエッジコンピューティングと、エッジで Vue.js を使ってサーバーサイドレンダリングを行う方法について学びます。そのために、3 つの異なる技術を見ていきましょう。

- JavaScript コードをエッジにデプロイするための破壊的なサーバーレスプラットフォームである「Cloudflare Workers」。
- Web アプリを開発・構築するための新しいフロントエンドツールである「Vite」。
- そして、私が現在取り組んでいる「Vitedge」というフレームワークを使って、エッジでウェブアプリを実行するために、これらをどのように組み合わせるか。

私と同じようにこのテーマに興味を持っていただき、トークを楽しんでいただければ幸いです。では、さっそく始めましょう。

# サーバーレスの誤解

まず最初に、文脈を説明します。皆さんは「サーバーレス」という言葉を聞いたことがあると思います。「クラウド機能」や「ラムダ」とも呼ばれています。私が数年前に初めてこの言葉を聞いたときは、自分の作った API が低レイテンシーかつ高いスケーラビリティを持って世界中で動作するようになると想像しました。実際には、この「サーバーレス」は、私が期待していたものの半分しか実現できませんでした。

確かにスケーラブルで、従量制の課金があるのは事実です。しかし、一般的には世界の 1 箇所（通常は米国）でしか稼働しません。

その上、これらのクラウド機能は「コールドスタート」と呼ばれるものに悩まされています。これは基本的に、私たちの関数が実行されてからしばらくすると、サーバーのメモリから削除される可能性があり、次のリクエストでは、サーバーが再びメモリに関数をロードする必要があることを意味しています。このプロセスは、このグラフ（画像参照）でわかるように、ある程度の時間がかかることがあります。AWS：最大で 0.5 秒、GCP：最大で 1.5 秒、Azure：最大で 4 秒です。

それとは別に、クラウドの機能はステートレスであり、そのメモリやデータは、一般的に異なるリクエスト間で共有することができません。これは、メモリリークやその他のエラーを防ぐことができるので、通常は良いことです。しかし、認証やユーザーセッションの保存など、「ステートフル」な環境に比べて作業が複雑になる場合もあります。このような場合には、Redis のような別のサービスが必要になります。

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

これらは、ワーカーが一般的に使用される例です。

- A/B テスト。ユーザーの位置情報などに基づいて、異なる情報を返すことができます。
- Anlytics。ある程度までは、SDK をブラウザにプッシュすることなく、Worker から直接、優れたアナリティクスを行うことができます。
- 最近しなければならなかったことがあります。日本のオンラインショップで、海外のお客様にだけバナーを表示することです。日本国内のユーザーには何も表示されませんが、海外のユーザーにはバナーが表示されます。

# Workers の欠点

So far we have seen a lot of benefits to using Cloudflare Workers. However, there are also some downsides:

- Workers tend to follow Web APIs instead fo Node.js APIs. This means that there are a lot of NPM packages for backend that are not supported in the Worker environment, such as Stripe SDK or Auth0's JWT verification utilities. We need to use raw APIs or find utilities that rely on Web Standards instead of Node.js APIs.
- Workers run in a very constrained environment. This means there are runtime limits such as maximum memory, CPU time, or the number of subrequests we can make. Not every application can run in this environment and we need to keep that in mind when building our apps.
- Also, the developer experience is not that great yet compared to other providers, which might make debugging a bit harder, but they are improving.
- The cumminity is still young so you might need to spend more time to learn how to do certain things.

Now that we have seen what actually "deploying to the edge" means, by using Cloudflare Workers, let's have a look at Vite.

# Vite

Vite という名前について聞いてことがあるかもしれません。Vite は Vue.js の作者 Evan You によって開発された新しいフロントエンドツールです。
Webpack そして Webpack-dev-server を組み合わせたものと考えていいですが、はるかに速く、そして設定も簡単です。

このおかげで、Viteを使ってWebアプリケーションを作成する際のDXはとても良いものになっています。デフォルトの設定値はほとんどのアプリケーションに対応していますし、設定を変更する必要がある場合でも、とても簡単です。例えば、SASSやSCSSを使用する必要がある場合は、それらのコンパイラをインストールするだけで、Viteは自動的にそれらを選択します。

Viteの開発サーバーはほとんど瞬時に開始します。そして、Hot Module Replacement (HMR)も同様です。ファイルを保存するとすぐにブラウザでアップデートが表示されます。

それに加えて、Viteのコミュニティはとても活発です。多くの優秀な開発者が、想像できるすべてのもののためにプラグインを作っています。ファイルシステムのルーティング、コンポーネントの自動インポート、国際化のためのプリコンパイル、アイコンへの簡単なアクセス、などなど。プラグインをインストールして、1行のコードでインポートするだけでいいのです。

Viteはフレームワークに依存がありませんが、もちろん Evan You によって作られているため、Vue.js で完璧に動作することは想像に難くありません。

それとは別に、私が最も気に入っているのは、Viteがとても簡単に拡張できることです。內部関数のほとんどをエクスポートしているので、WebSocketやサーバーサイドレンダリングを扱うツールを簡単に作ることができます。

そして、これこを私がこの数ヶ月間取り組んできたことなのです。Vite SSR ツールをエッジで Cloudflare Workersで実行しています。

# Vitedge: エッジで動作する SSR (1)

そして、これを私は Vitedge と呼んでいますが、Vite アプリケーションを作ってそれを Couldflare Workers にデプロイすると、ページコンポーネントの HTML をエンドユーザーに近いエッジでレンダリングすることができます。複雑に聞こえるかもしれませんが、それはシングルページアプリケーションの開発するとき同じくらい簡単です。

したがって、CDN から静的な`index.html`をアップロードして提供する代わりに、CND ノード自身が`index.html`ファイルをレンダリングし、次のリクエストのためにエッジにキャッシュします。そのため、次のリクエストでは、あたかも静的なファイルであるかのようにファイルを取得します。

# Vitedge: エッジで動作する SSR (2)

おそらく、以下のようないくつかのメリットを想像できます:

- より良いロード時間、遅いビルドなしによる SEO、動的なコンテンツをサポート
- 素晴らしいパフォーマンス
- 構成可能なキャッシュ、つまり、私達の（データベースやコンテンツ管理システムからの）データが特定のルートで変更されたことがわかっている場合は、そのルートをキャッシュから削除し、他のルートはそのままにしておくことができるのです。そのため、そのルートに対する次のリクエストは、新しいデータを使用することになります。
- Vitedge はファイルシステムによるルートに基づいた API エントリポイントを作成する方法も提供します。
- そして、もちろん、Vite がもたらす良い DX もすべてあります。

SSR は多くのアプリケーションに適しているかもしれまんが、特定のプロジェクトにおいては適していないかもしれないということも、私は伝えておきたいと思います。プロジェクトによっては、SPA と静的サイトジェネレーターは良い選択となるでしょう。

それでは、Vite アプリケーションに Vitedge をインストールして使用する方法を見てみましょう。

# Vitedge: エッジで動作する SSR (3)

もし、まだ Vite に慣れていない方は、通常のアプリは次のようになっています。`vite.config`ファイルと Vue.js アプリケーション向けの通常の"source"フォルダがあり、ページコンポーネント、ルート、ルートコンポーネント(App)、そして Vue アプリ用のメインエントリファイルがあります。

Vitedge をインストールする方法は、単純にそのプラグインを Vite の設定ファイルに追加し、Vue アプリケーションのエントリポイントを次のように変更します。Vitedge は環境に応じて、`createApp`または`createSsrApp`を呼び出します。このメインフックでは、i18n、Vuex、Pinia などあらゆる Vue.js モジュールをセットアップすることができます。

デフォルトでは、Vue の SPA のように、Vitedge ではクライアントそしてサーバー向けに 1 つのエントリポイントを持つ代わりに 1 つのエントリポイントを持ちます。これにより SSR を簡単に使い始めることができます。ただ、必要に応じて 2 つの独立したエントリポイントを使用することもサポートされています。

# Vitedge: エッジで動作する SSR (4)

それでは、API からデータを取得するページコンポーネントの作成方法について説明します。
Vitedge では、フロントエンド向けに`source`フォルダの隣にある`functions`というフォルダに API を配置します。

ここでは、ブログ記事を表示するためのシンプルなページコンポーネントを用意し、"post"というルートに"/posts/:slug"というパスで配置しています。
props からブログポストデータを取得するために、ページコンポーネントを期待しているのが分かります。そして、それから`vueuse/head`を使っていくつかの meta タグを書くためにそのデータを使い、DOM にいくつかのコンテンツを表示しています。

さて、この Post オブジェクトをどのようにしてページコンポーネントに提供するのでしょうか？覚えていると思いますが、このページのルートは `post`と呼ばれています。"functions/props/"ディレクトリ配下に、ルートと同じ名前の新しいファイル "functions/props/post.ts"（TS または JS）を作成する必要があります。この関数は、ブラウザからこのルートにアクセスするたびに呼び出され、その結果を props としてページコンポーネントに渡します。ルートは"slug"パラメータを期待しており、これは実際に props 関数の引数で提供されていることがわかります。

ここでは、この"slug"パラメータを使って、データベースやコンテンツマネジメントシステムから`fetch`で投稿情報を取得し、その結果を返すことができます。

とてもシンプルですね。この関数コードは Web アプリケーションにバンドルされていないので、API キーなどのプライベートな情報をここに持つことができます。また、ここではキャッシュ値を指定していることに注目してください。これは、このページコンポーネントのレンダリングされた HTML が、この秒数の間、エッジにキャッシュされることを意味します。この記事の CMS や DB のデータが変更されたことに気づいたら、Cloudflare のキャッシュにリクエストしてこの情報を削除し、次のリクエストで新しいデータを使用できるようにします。

ここには、`.env`や sitemap といったファイルがあることに注目してください。
`.env`ファイルには、API で利用可能な環境変数が格納されています。一方、sitemap ファイルは、この関数(props ハンドラ)のようなものですが、ブラウザで`/sitemap.xml`や`/sitemap.txt`にアクセスしたときに呼び出されるので、DB や CMS のデータに依存した動的なサイトマップを返すことができます。もちろん、キャッシュにも対応しています。サイトマップの代わりに、`robots`ファイルや、GraphQL サーバーをセットアップする`graphql`ファイルを用意することもできます。

今日の発表は以上になります。Vitedge についてもっと知りたい方は、https://vitedge.js.org をご覧ください。
