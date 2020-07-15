# Update: Simple (T)OTP Server for Nginx Auth (Win10x64)
Download: [SimpleOTP_AllInOne_Win10x64_C$.zip](https://github.com/yu2n/simpleotp/releases/download/0.1/SimpleOTP_AllInOne_Win10x64_C.zip)

```
How To Use:

1. Download "SimpleOTP_AllInOne_Win10x64_C$.zip", Expand To "C:\"

2. Run "C:\nginx-1.19.1\nginx.exe"
    Tips: Edit "C:\nginx-1.19.1\conf\nginx.conf" to more ..

3. Run "C:\SimpleOTP\SimpleOTP-Server-Start.bat"

4. Run Chrome, GoTo  http://localhost/ , Your See It ...
```

```
nginx/Windows-1.19.1
https://nginx.org/en/download.html

Python38  (pip install pyotp)
https://www.python.org/downloads/

SimpleOTP 
https://github.com/newhouseb/simpleotp  (test fail, ver.0)
https://github.com/toastie89/simpleotp  (test ok)
https://github.com/yu2n/simpleotp  (my ver, for win10)

2020.07.14 YY 
```

---


# Super Basic TOTP `auth_request` Server for nginx

## What is this for?

Have you ever wanted to add more security to a web application without modifying the web application itself? Take for example Jupter Notebook/Lab, which allows you to run arbitrary code from a web browser. It supports a built-in password / token-based authentication. Hopefully you're using a unique password, but if you're following proper security practices it's generally a good idea to protect stuff with "something you know and something you have." Chances are that if you've gotten this far you don't need me to convince you of the merits of two factor authentication.

## How does it work?

I use nginx in front of a variety of web services to handle SSL termination (using letsencrypt, which is amazing and you should also use). Nginx has a handy module called auth_request that you can use to specify an endpoint to check if a user is authenticated. If the endpoint returns 200, the parent request is allowed to succeed, otherwise a 401 error is returned. You can set up nginx to then redirect the user to a login page where they can do whatever they need to assert proof of identity.

In this case, the auth endpoint is reverse proxied to the simple script in this repo, which does things like token checking and presenting a login form.

## Example Configuration

In something like `/etc/nginx/sites-enabled/default`

```
server {
        server_name jupyter.example.com;

        location /auth {
                proxy_pass http://127.0.0.1:8000; # This is the TOTP Server
                proxy_set_header X-Original-URI $request_uri;
        }

        # This ensures that if the TOTP server returns 401 we redirect to login
        error_page 401 = @error401;
        location @error401 {
            return 302 /auth/login;
        }

        location / {
                auth_request /auth/check;
                proxy_pass http://127.0.0.1:8888; # This is Jupyter

                # This is needed for Jupyter to proxy websockets correctly, 
                # it's unrelated to auth but handy to have written down here 
                # for reference anyhow...
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
        }

# The rest of the server definition, including SSL and whatnot
```

## Additional assembly required:

1. You need to run `main.py` with Python3.5+ in a tmux session or something like supervisord.
2. You should generate a TOTP secret (i.e. `import pyotp; print(pyotp.random_base32())`) and store it in `.totp_secret` alongside `main.py` and also your two factor auth manager of choice (Google Authenticator, Duo, etc.)

## FAQ

**Wait, this checks the TOTP secret before you enter a password?**

Yep, it feels kinda backwards, but I only have one login anyhow and I've rate-limited TOTP checks, so you can't hammer auth to figure out the TOTP secret.

**What about CSRF attacks?**

In my case Jupyter already prevents CSRF attacks and the only thing you could do (as far as I can tell) with CSRF attacks on the auth server is log a user out, so I haven't bothered.
