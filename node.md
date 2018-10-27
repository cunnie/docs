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
make the following changes:
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
