```
brew install nginx
brew services start nginx # starts at login
less /usr/local/etc/nginx/servers/*
nginx # start ad hoc
vim /usr/local/etc/nginx/nginx.conf
  root /Users/cunnie/workspace/sslip.io/document_root
nginx -g "daemon off;" # runs in foreground
```
