Create a new node project

```bash
npm init
```

Add a dependency (e.g. lite-server)

```bash
npm install lite-server --save-dev
```

Create an entrypoint:

```bash
vim package.json
```

Make the following changes:

```diff
   "scripts": {
-    "test": "echo \"Error: no test specified\" && exit 1"
+    "start": "npm run lite",
+    "test": "echo \"Error: no test specified\" && exit 1",
+    "lite": "lite-server"
   },
```

Run your app:

```bash
npm start
```

Note that `npm start` will run whatever is in your `.scripts.start` of your
`package.json`; `.main` is the entrypoint. For example, let's say `.main`
`index.html`, and these are its contents:

```html
<html>
<title>Dawgs</title>
<body>
	<h1>I love muh dawg!</h1>
	<p>I really do</p>
	<h2>I love muh cat!</h2>
	<p>but not as much as muh dawg!</p>
</body>
</html>
```

In that case, the above HTML is displayed on <http://localhost:3000> when `npm
start` is run.
