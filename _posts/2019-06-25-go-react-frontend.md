---
title: "Creating a React Frontend for a Go API"
layout: post
date: 2019-06-25
headerImage: false
projects: false
hidden: false # don't count this post in blog pagination
tag:
- golang
- programming
- api
- webapp
- tutorial
- react
- javascript
- frontend
- accessibility
category: blog
author: rshipp
description: Add a React frontend with AJAX calls on top of a Go web API, and accessibility testing with axe. Continued from "Building a Go Web API."
---

I'm learning Go (and React!) by building a small API-backed web application, and wanted to share the process in case it helps someone else. In this post, we'll continue where we left off last time with the [Go web API][1] for managing [GitHub stars][2], by setting up the beginnings of a React frontend with AJAX loading from the API. We'll also tie in [axe][3] for accessibility testing, to make sure everyone is able to use our app. If you'd like to follow along with this post without going through the previous ones, you can grab a copy of the API (`main.go`) from [this GitHub repo][4].

What You'll Need
----------------

Before we get started, you'll need a few things:

* [Node.js][5] installed on your computer.
* [Go][6] installed on your computer.
* The [Go web API][4] (`main.go`) from this [previous post][1].
* A text editor.

Feel free to skim through the [React tutorial][7] or another introduction to React if you haven't already, but you won't need it for this initial setup.

If you run into anything unclear in this post, feel free to [open an issue][8] on GitHub and let me know!

Background
----------

There are a lot of ways to start building a React app. We're going to go the easy route and use [create-react-app][9], a set of scripts and preconfigured build tools that will get us up and running without needing to learn a bunch of extra tools first.

Full disclosure: I started out trying to do everything by hand, because it seemed to me like `create-react-app` was hiding complexity at the cost of increased overhead and decreased flexibility, which I don't generally like. I eventually got everything working, and learned a lot in the process, but by the end I realized that what I'd made was not going to scale well as I continued building out the app. I considered leaving it alone and learning how to use build tools like grunt and webpack to clean up some of the issues I knew would come up. But there are a lot of options here, a lot of ways to make small mistakes that will cost us later on, and even if we do everything "right," we still have to maintain each dependency we introduce ourselves - not a small task.

More than a quick-start app creator, what `create-react-app` really gives us is an officially maintained, continuously updating, and rigorously tested scaffold to build on. It means we can focus on writing our app, and learning React, instead of wading through the entire JavaScript ecosystem trying to solve problems we haven't even encountered yet. And of course, if we ever outgrow it, we can always [eject][34] our app and start managing all the build tools manually.

Initial Setup
-------------

We'll start by creating a new React app, in the same folder as `main.go`:

```shell
npx create-react-app star-manager
```

The `npx` command executes [npm][10] package binaries, downloading and installing them on-the-fly as needed. In this case we're executing `create-react-app`, with the name of our new app: `star-manager`.

We should now have a directory structure like this:

```
main.go
star-manager/public/
star-manager/src/
<...>
```

If you'd like to keep your React frontend and Go backend separate, you can leave the `star-manager` folder as-is. Or, you could rename `star-manager` to `frontend`, and move `main.go` into a new `backend` folder. I chose to merge the two together, and move everything from `star-manager` up into my project root:

```shell
# On Linux/OSX:
mv star-manager/{src,public,package*,node_modules} ./
rm -rf star-manager/
```

The result looks something like this:

```
main.go
public/
src/
<...>
```

How you structure your project is up to you, and it's totally fine to change it as the project grows! Like the official React docs say, [don't overthink it][11].

Let's take a look at what `create-react-app` gave us.

* `package.json`: Project metadata (name, version, etc), dependencies (only `create-react-app` for now), and build instructions.
* `package-lock.json`: Specific versions of every JavaScript package installed, managed my `npm`, so someone else can reproduce our environment exactly.
* `node_modules/`: JavaScript packages used to build our app, managed by `npm`. We don't need to worry about this.
* `public/`: Static files like HTML and images. 
* `src/`: JavaScript and CSS source code that runs the dynamic parts of our app. This is where we'll spend most of our time.

Running `npm start` brings up a welcome screen with a spinning React logo in our browser. We can go through and clean out everything we don't need in `src/` and `public/` (delete images, empty CSS files, etc) or just leave them as-is and replace their contents as we go.

To tie in our Go API, we just need a couple changes to `package.json`:

```json
  "scripts": {
    "start": "react-scripts start",
    "server": "go run main.go",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "proxy": "http://localhost:8080",
```

Adding a new `server` command to the `scripts` section lets us start the API server with `npm run server`. Adding the `proxy` field tells the JavaScript development server to [send API requests to our backend][12] instead of just returning a 404 when they don't match any of the pages defined in the React frontend.

Now, when we want to test the full app, we have to leave `npm run server` running in the background, and start the React development server with `npm start`.

Loading From the API
--------------------

[AJAX][13] is an (outdated) acronym for a commonly used technique for loading information [asynchronously][14], in the background of an application. Any type of dynamic loading on a webpage is [usually][15] done using AJAX.

In the past, I've always used jQuery's [$.ajax()][16] function to make AJAX calls, since it's much easier to work with than the old browser built-in [XMLHttpRequest][17]. But evidently somebody [made fetch happen][18], and we now have global access to a wonderful, [promise][19]-based async HTTP function! We'll be using `fetch()` in our app to load information from our backend API into the React frontend.

There's a great page on [using AJAX with React][20] in the official React FAQs. We can copy their example code and use it in our app with just a little modification. In `src/App.js`:

```jsx
import React from 'react';
import './App.css';

class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      error: null,
      isLoaded: false,
      items: []
    };
  }

  componentDidMount() {
    fetch("/stars")
      .then(res => res.json())
      .then(
        (result) => {
          this.setState({
            isLoaded: true,
            items: result
          });
        },
        // Note: it's important to handle errors here
        // instead of a catch() block so that we don't swallow
        // exceptions from actual bugs in components.
        (error) => {
          this.setState({
            isLoaded: true,
            error
          });
        }
      )
  }

  render() {
    const { error, isLoaded, items } = this.state;
    if (error) {
      return <div>Error: {error.message}</div>;
    } else if (!isLoaded) {
      return <div>Loading...</div>;
    } else {
      return (
        <div>
          <h1>StarManager</h1>
          <ul>
            {items.map(item => (
              <li key={item.id}>
                <a href={item.url}>{item.name}</a> {item.description}
              </li>
            ))}
          </ul>
        </div>
      );
    }
  }
}

export default App;
```

When the `fetch()` API call gets a response, it populates `this.state` in our `App` class, and React knows to use the `render()` method to update the page with the newly returned items.

The code that looks like HTML inside the `render()` method is actually [JSX][21], an optional extension to React that makes it easier to write (and read) React [components][22]. JSX is compiled down to JavaScript by a tool called babel, which is included and preconfigured by create-react-app. We'll be using JSX extensively in our app.

The `import` statements at the top of the file and the `export` at the bottom are part of the [ES6 modules][23] specification. You can read more about [importing and exporting components][24] in the create-react-app docs.

If we run `npm start` (with `npm run server` still running in the background), we'll see our new `App` component loading information from the Go API server. (If you don't see any "items", you can insert some stars into the `test.db` SQLite database or POST to `/stars` to create them. More information on that [in the previous post][1].)

That's all we need for a fully functioning API-backed React app.

Accessibility
-------------

Accessibility, often shortened to [a11y][25], is the practice of making sure the things we build can be used by everyone. There's a very helpful page on [accessibility in the React docs][26], with guidelines and tools to help create and test accessible webapps. We'll be using [react-axe][27] to test and report on accessibility in the browser throughout development, as well as testing manually with [screen readers][28] and [keyboard-only navigation][29] later on.

Install `react-axe` with:

```shell
npm install --save-dev react-axe
```

The `--save-dev` flag tells `npm` this is a "development dependency," and won't be used when we publish to [production][30]. Since we're only using axe to find and fix accessibility issues as we build the app, we don't need to include it in public, production builds. Notice that `package.json` has a new `devDependencies` key with `react-axe` listed, and `package-lock.json` has been updated with the names and versions of the newly installed packages (`react-axe` and its dependencies).

Now that `react-axe` is installed, we can set it up in our app's [entry point][31], which for apps made with `create-react-app` is `src/index.js`:

```jsx
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
import App from './App';

// Use axe a11y testing in development.
var axe = require('react-axe');

if (process.env.NODE_ENV !== 'production') {
  axe(React, ReactDOM, 1000);
}

ReactDOM.render(<App />, document.getElementById('root'));
```

The `process.env.NODE_ENV` check tests the value of the `NODE_ENV` [environment variable][32] at build time, and disables axe in production builds. (We can get a production-ready build with run `npm run build`.) The `axe` call passes in the two React references, and the number of milliseconds to wait for the app to load before running the accessibility checks - in this case `1000` (1 second).

Let's fire up the React app again and see what axe reports:

```shell
npm start
```

When we open the development console in our browser, we see there are only a few medium-priority issues:

```
index.js:154 New aXe issues
index.js:180 moderate: Document must have one main landmark https://dequeuniversity.com/rules/axe/3.2/landmark-one-main?application=axeAPI
index.js:180 moderate: Page must contain a level-one heading https://dequeuniversity.com/rules/axe/3.2/page-has-heading-one?application=axeAPI
index.js:180 moderate: All page content must be contained by landmarks https://dequeuniversity.com/rules/axe/3.2/region?application=axeAPI
```

Axe helpfully provides links to learn more about the issues it's reporting, which in this case point to us having no `main` element in the page to tell a screen reader where the main content is.

It says the best way to fix this is by adding `<main role="main">`, so let's open up `public/index.html` and make sure all our main content is wrapped in that element:

```html
  <body>
    <main role="main">
      <noscript>You need to enable JavaScript to run this app.</noscript>
      <div id="root"></div>
      <!--
        This HTML file is a template.
        If you open it directly in the browser, you will see an empty page.

        You can add webfonts, meta tags, or analytics to this file.
        The build step will place the bundled scripts into the <body> tag.

        To begin the development, run `npm start` or `yarn start`.
        To create a production bundle, use `npm run build` or `yarn build`.
      -->
    </main>
  </body>
```

As we add more sections to our app (sidebars, headers, footers, etc), we'll have to revisit this and move `main` somewhere else to make sure only the "main" content is inside it.

Our React app automatically reloads when we change anything, so we can check the console again to see what axe says about our new `index.html`. It turns out all three of the reported issues have been fixed with that one change, and the console is empty! This doesn't necessarily mean there aren't any accessibility issues, but it is a great start. We can keep checking the console throughout development to catch new issues immediately, and periodically use some of the manual tests described in the React docs to verify everything works as intended.

That completes the setup for our new React frontend!

You can check out the complete code for this post [on GitHub][33].

Conclusion
----------

A quick recap:

1. We started a new React app with `create-react-app`.
2. Added an AJAX call to our backend API using `fetch()`.
3. Integrated `react-axe` for in-browser accessibility testing, and fixed an issue it reported.

In the process, we touched on several React and JavaScript features, and web applications concepts:

* Asynchronous loading with AJAX and `fetch()`.
* ES6 features: promises and modules.
* `npm`, `npx`, and `package.json`.
* Basic `create-react-app` usage.
* Using JSX in React components.
* Proxying requests from the JavaScript development server to a third-party backend with `create-react-app`.
* Developing with accessibility in mind, and testing with axe.
* And more!

In future posts, I'll revisit this React app and the backend Go API, and continue adding new functionality until the star app is complete.

[1]: https://rshipp.com/go-web-api
[2]: https://help.github.com/en/articles/about-stars
[3]: https://www.deque.com/axe/
[4]: https://github.com/rshipp/StarManager/tree/3394f15493add8ed74b98f4688c5963cdd4de69d
[5]: https://nodejs.org/en/download/
[6]: https://golang.org/dl/
[7]: https://reactjs.org/tutorial/tutorial.html
[8]: https://github.com/rshipp/rshipp.github.io/issues
[9]: https://facebook.github.io/create-react-app/
[10]: https://en.wikipedia.org/wiki/Npm_(software)
[11]: https://reactjs.org/docs/faq-structure.html#dont-overthink-it
[12]: https://facebook.github.io/create-react-app/docs/proxying-api-requests-in-development
[13]: https://www.w3schools.com/xml/ajax_intro.asp
[14]: https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Concepts
[15]: https://www.html5rocks.com/en/tutorials/websockets/basics/
[16]: https://api.jquery.com/jQuery.ajax/
[17]: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
[18]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch
[19]: https://www.tutorialspoint.com/es6/es6_promises.htm
[20]: https://reactjs.org/docs/faq-ajax.html
[21]: https://reactjs.org/docs/introducing-jsx.html
[22]: https://reactjs.org/docs/components-and-props.html
[23]: http://exploringjs.com/es6/ch_modules.html
[24]: https://facebook.github.io/create-react-app/docs/importing-a-component
[25]: https://a11yproject.com/posts/a11y-and-other-numeronyms/
[26]: https://reactjs.org/docs/accessibility.html
[27]: https://github.com/dequelabs/react-axe
[28]: https://reactjs.org/docs/accessibility.html#screen-readers
[29]: https://reactjs.org/docs/accessibility.html#the-keyboard
[30]: https://en.wikipedia.org/wiki/Development,_testing,_acceptance_and_production
[31]: https://webpack.js.org/concepts/entry-points/
[32]: https://en.wikipedia.org/wiki/Environment_variable
[33]: https://github.com/rshipp/StarManager/tree/eeaa452a6a5a76b10ffb9652a7cfb0c518fe3e05
[34]: https://facebook.github.io/create-react-app/docs/available-scripts#npm-run-eject
