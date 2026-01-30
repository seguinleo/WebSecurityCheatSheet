# WebSecurityCheatSheet

## 📖 Table of contents

- [**HTTPS (i.e. Apache2)**](#https-ie-apache2)
  - [**Let's Encrypt**](#lets-encrypt)
  - [**Redirect all HTTP traffic to HTTPS**](#redirect-all-http-traffic-to-https)
  - [**Transport Layer Security (TLS)**](#transport-layer-security-tls)
  - [**HTTP Strict Transport Security (HSTS)**](#http-strict-transport-security-hsts)
  - [**SSL**](#ssl)
- [**Apache2**](#apache2)
  - [**Enable HTTP2**](#enable-http2)
  - [**Enable mod\_security**](#enable-mod_security)
  - [**Hide server signature**](#hide-server-signature)
  - [**Restrict access to files**](#restrict-access-to-files)
- [**Nginx**](#nginx)
- [**Database**](#database)
- [**Authorization**](#authorization)
- [**Cookies**](#cookies)
- [**CSRF Token**](#csrf-token)
- [**PHP**](#php)
  - [**PHP-FPM**](#php-fpm)
  - [**PHP PDO**](#php-pdo)
  - [**php.ini**](#phpini)
- [**Express.js**](#expressjs)
- [**User login**](#user-login)
- [**Node.js/npm**](#nodejsnpm)
- [**Docker**](#docker)
- [**Ubuntu VPS**](#ubuntu-vps)
- [**HTTP Headers**](#http-headers)
- [**HTML DOM sanitization**](#html-dom-sanitization)
  - [**Links**](#links)
  - [**POST vs GET**](#post-vs-get)
  - [**e.innerHTML**](#einnerhtml)
  - [**eval() and new Function()**](#eval-and-new-function)
  - [**DOMPurify**](#dompurify)
- [**Analysis tools**](#analysis-tools)
- [**Sources and resources**](#sources-and-resources)

**_This document is a concise guide that aims to list the main web vulnerabilities, particularly JavaScript, and some solutions. However, it is exhaustive and should be supplemented with quality, up-to-date documentation._**

**_This guide is intended for full-stack developers working with JavaScript technologies (React, Vue, etc.) and a Node.js/PHP backend with CRUD/REST APIs._**

**_.NET, JAVA, Django or Ruby are therefore not included in this guide._**

## **HTTPS (i.e. Apache2)**

### **Let's Encrypt**

Install certificates with Let's Encrypt:

```bash
sudo apt install certbot python3-certbot-apache
```

```bash
sudo certbot certonly --standalone -d example.com -d www.example.com
```

Add a CAA record to your DNS zone:

```bash
CAA 0 issue "letsencrypt.org"
```

### **Redirect all HTTP traffic to HTTPS**

Write in _/etc/apache2/apache2.conf_:

```apache
Redirect permanent / <https://domain.com/>
```

### **Transport Layer Security (TLS)**

General purpose web applications should default to TLS 1.3 (support TLS 1.2 if necessary) with all other protocols disabled. Only enable TLS 1.2 and 1.3. Go to _/etc/apache2/conf-available/ssl.conf_ and write:

```apache
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
```

### **HTTP Strict Transport Security (HSTS)**

HTTP Strict Transport Security (HSTS) is a mechanism for websites to instruct web browsers that the site should only be accessed over HTTPS. This mechanism works by sites sending a Strict-Transport-Security HTTP response header containing the site's policy. Write in _/etc/apache2/apache2.conf_:

```apache
Header set strict-transport-security "max-age=31536000; includesubdomains; preload"
```

Reload Apache and submit your website to <https://hstspreload.org/>

### **SSL**

Disable all non-secure encryption algorithms. Go to _/etc/apache2/conf-available/ssl.conf_ and write:

```apache
SSLCipherSuite ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384
```

Online Certificate Status Protocol stapling is a crucial technology that enhances both the speed and privacy of SSL/TLS connections. Go to _/etc/apache2/conf-available/ssl.conf_ and write:

```apache
SSLUseStapling on
SSLStaplingResponderTimeout 5
SSLStaplingReturnResponderErrors off
SSLStaplingCache "shmcb:ssl_stapling(32768)" <- before VirtualHost
```

## **Apache2**

### **Enable HTTP2**

HTTP/2 provides a solution to several problems that the creators of HTTP/1.1 had not anticipated. In particular, HTTP/2 is much faster and more efficient than HTTP/1.1:

``sudo a2enmod http2``

### **Enable mod_security**

Mod security is a free Web Application Firewall (WAF) that works with Apache2 or nginx:

``sudo apt install libapache2-modsecurity``

```apache
SecRuleEngine On <- /etc/modsecurity/modsecurity.conf
```

### **Hide server signature**

Revealing web server signature with server/PHP version info can be a security risk as you are essentially telling attackers known vulnerabilities of your system. Write in _/etc/apache2/apache2.conf_:

```apache
ServerTokens Prod
ServerSignature Off
```

### **Restrict access to files**

Write in _/etc/apache2/apache2.conf_:

```apache
<Directory />
  Options FollowSymLinks
  AllowOverride None
  Require all denied
</Directory>
<Directory /var/www>
  Options -Indexes
  AllowOverride None
  Require all granted
</Directory>
```

## **Nginx**

```nginx
server {
  listen 80;
  server_name localhost;
  server_tokens off;
  proxy_hide_header X-Powered-By;

  location / {
    root /usr/share/nginx/html;
    index index.html;
    try_files $uri $uri/ /index.html;

    limit_except GET POST {
      deny all;
    }
  }
}
server {
  listen 443 ssl http2;

  ssl_stapling on;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
  ssl_prefer_server_ciphers on;
}
```

## **Database**

- Use a strong database password and restrict user permissions
- Hash all user login passwords before storing them in the database
- For MySQL/MariaDB databases, use prepared queries to prevent injections

```php
# php
$query = $PDO->prepare("SELECT a FROM b WHERE c=:c LIMIT 1");
$query->execute([':c' => $c]);
$row = $query->fetch();
```

```js
// js
try {
  const [rows] = await pool.execute(
    "SELECT a FROM b WHERE c = ? LIMIT 1",
    [userId]
  )

  if (rows.length !== 1) return 0
  return rows[0].a
} catch {
  return 0
}
```

- For MySQL/MariaDB databases, use _mysql_secure_installation_
- For NoSQL databases, like MongoDB, use a typed model to prevent injections
- Avoid _$accumulator_, _$function_, _$where_ in MongoDB
- Use .env for database and server secrets, encrypt it with _dotenvx_
- Encrypt all user data (e.g. AES-256-GCM), store encryption keys in a secure vault like AWS Secrets Manager, Google Secrets Manager or Azure KeyVault

## **Authorization**

- Deny by default
- Enforce least privileges
- Validate all permissions
- Validate files access
- Sanitize files upload
- Require user password for sensitive actions

## **Cookies**

``Domain=domain.com; Path=/; Secure; HttpOnly; SameSite=Lax or Strict``

**Secure:** All cookies must be set with the _Secure_ directive, indicating that they should only be sent over HTTPS

**HttpOnly:** Cookies that don't require access from JavaScript should have the _HttpOnly_ directive set to block access

**Domain:** Cookies should only have a _Domain_ set if they need to be accessible on other domains; this should be set to the most restrictive domain possible

**Path:** Cookies should be set to the most restrictive _Path_ possible

**SameSite:**

- **Strict (preferred):** Only send the cookie in same-site contexts. Cookies are omitted in cross-site requests and cross-site navigation
- **Lax:** Send the cookie in same-site requests and when navigating to your website. Use this value if _Strict_ is too restrictive

> [!IMPORTANT]
> Since using SameSite with the **Strict** attribute is relatively safe, it is also recommended to use a CSRF token

## **CSRF Token**

```js
// js/node
const verifyCsrfToken = (req, res, next) => {
  const userToken = req.headers['x-csrf-token']
  const storedToken = req.cookies?.csrfToken

  if (!userToken || !storedToken) {
    return res.status(403).json({ error: 'Invalid CSRF token' })
  }

  try {
    const userBuffer = Buffer.from(userToken, 'utf8')
    const storedBuffer = Buffer.from(storedToken, 'utf8')

    if (userBuffer.length !== storedBuffer.length || !crypto.timingSafeEqual(userBuffer, storedBuffer)) {
      return res.status(403).json({ error: 'Invalid CSRF token' })
    }
  } catch {
    return res.status(403).json({ error: 'Invalid CSRF token' })
  }
  next()
}
```

## **PHP**

### **PHP-FPM**

PHP-FPM (FastCGI Process Manager) is often preferred over Apache mod_php due to its superior performance, process isolation, and flexible configuration:

```bash
sudo apt install php<version>-fpm
sudo a2dismod mpm_prefork
sudo a2enmod mpm_event proxy_fcgi proxy
```

### **PHP PDO**

PDO (PHP Data Objects) is a Database Access Abstraction Layer that provides a unified interface for accessing various databases. A secure MySQL database connection with PDO:

```php
$options = [
  PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
  PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
  PDO::ATTR_EMULATE_PREPARES   => false,
];
$dsn = "mysql:host=$host;dbname=$db";
try {
  $PDO = new PDO($dsn, $user, $pass, $options);
} catch (Exception $e) {
  throw new Exception('Connection failed');
  return;
}
```

### **php.ini**

A hardened template for PHP-FPM, write in _/etc/php/&lt;version&gt;/fpm/php.ini_:

```ini
expose_php               = off
error_reporting          = e_all & ~e_deprecated & ~e_strict
display_errors           = off
display_startup_errors   = off
ignore_repeated_errors   = off
allow_url_fopen          = off
allow_url_include        = off
session.use_strict_mode  = 1
session.use_only_cookies = 1
session.cookie_secure    = 1
session.cookie_httponly  = 1
session.cookie_samesite  = strict
session.sid_length       = > 128
```

## **Express.js**

Secure Express.js with HTTPS:

```js
https
  .createServer({
    key: fs.readFileSync(process.env.SSL_KEY),
    cert: fs.readFileSync(process.env.SSL_CERT),
  }, app)
  .listen(PORT, () => {
    console.log('Server listening on: https://localhost:%s', PORT)
  })
```

Secure Express.js with Helmet and remove `x-powered-by`:

```js
app.use(helmet())
app.disable('x-powered-by')
```

Secure all routes with express-rate-limit:

```js
const limiter = rateLimit({
  windowMs: 5 * 60 * 1000,
  max: 100,
  message: 'Too many requests, please try again later.',
})

app.use(limiter)
```

Secure sessions with express-session:

```js
app.use(session({
  store: redisStore,
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    maxAge: 604800000,
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'Strict'
  }
}))
```

Verify authentication with JWT tokens:

```js
/**
  * This code is a middleware function designed to authenticate users in a web application using a JWT (JSON Web Token).
  * It first checks if the user ID and JWT token exist in the request and are properly formatted. Then, it verifies whether the token has been revoked by checking a blacklist stored in Redis (ie logout). After that, it decodes and validates the token’s signature to ensure it is genuine and belongs to the logged-in user.
  * If everything checks out, the request is allowed to proceed. If any step fails, the function returns an unauthorized error.
  */
const verifyJWTToken = async (req, res, next) => {
  const token = req.cookies?.jwtToken
  if (!token) return res.status(403).json({ response: 0 })

  try {
    const isBlacklisted = await redisClient.get(`blacklist:${token}`)
    if (isBlacklisted) return res.status(403).json({ response: 0 })

    let decoded
    try {
      decoded = jwt.verify(token, process.env.JWT_SECRET)
    } catch {
      return res.status(403).json({ response: 0 })
    }

    const tokenActive = await redisClient.sIsMember(`user:tokens:${decoded.id}`, token)
    if (!tokenActive) return res.status(403).json({ response: 0 })

    req.userId = decoded.id

    next()
  } catch {
    return res.status(403).json({ response: 0 })
  }
}
```

## **User login**

A secure express.js login system using JWT tokens, rate limiting and passport.js:

```js
/**
 * A secure login with JWT Token and CSRF Token using local passport.js
 */
import rateLimit from 'express-rate-limit'

const loginLimiter = rateLimit({
  windowMs: 3 * 60 * 1000,
  max: 5,
  message: 'Too many login attempts from this IP, please try again later'
})

app.post('/login', loginLimiter, async (req, res, next) => {
  try {
    const user = await new Promise((resolve, reject) => {
      passport.authenticate('local', { session: false }, (err, user) => {
        if (err) return reject(err)
        resolve(user)
      })(req, res, next)
    })

    if (!user) return res.status(401).send('Wrong username or password.')

    const token = jwt.sign(
      { id: user.id },
      process.env.JWT_SECRET,
      { expiresIn: 604800 }
    )

    await redisClient.sAdd(`user:tokens:${user.id}`, token)
    await redisClient.expire(`user:tokens:${user.id}`, 604800)

    req.session.regenerate((err) => {
      if (err) return res.status(401).send('Internal server error')

      req.session.userId = user.id

      res.cookie('jwtToken', token, {
        maxAge: 604800000,
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'Strict'
      })

      const csrfToken = crypto.randomBytes(32).toString('hex')
      res.cookie('csrfToken', csrfToken, {
        maxAge: 604800000,
        httpOnly: false,
        secure: true,
        sameSite: 'Strict'
      })

      return res.status(200).send('Logged in!')
    })
  } catch {
    return res.status(401).send('Wrong username or password.')
  }
})
```

## **Node.js/npm**

- Always keep all npm dependencies up to date
- Limit the use of dependencies
- Use _npm doctor_ to ensure that your npm installation has what it needs to manage your JavaScript packages
- Use eslint to write quality code
- To manage user cookies, use express.js and passport.js with JWT tokens

## **Docker**

- Use official and minimal images
- Use _.dockerignore_ to hide server secrets
- Run containers with a read-only filesystem using _--read-only_ flag
- Avoid the use of _ADD_ in favor of _COPY_
- Set a user with restricted permissions in _DockerFile_

```dockerfile
RUN groupadd -r myuser && useradd -r -g myuser myuser
# HERE DO WHAT YOU HAVE TO DO AS A ROOT USER LIKE INSTALLING PACKAGES ETC.
USER myuser
```

## **Ubuntu VPS**

- Use a strong passwords for all users
- Disable root login
- Create a user with restricted permissions and 2FA or physical key
- Always update all packages and limit their number
- Disable unused network ports
- Change SSH port and use Fail2Ban to prevent DoS and Bruteforce attacks, disable SSH root login in _sshd_config_

```ini
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

- Always make secure and regular backups
- Log everything
- Use SFTP instead of FTP
- Use a firewall like iptables or ufw
- Use _robots.txt_ to disallow all by default and don't disclose sensitive URLs

```ini
User-agent: \*
Disallow: /admin <- don’t do this
```

## **HTTP Headers**

A hardened template for Apache2 and nginx:

```apache
x-content-type-options: "nosniff"
access-control-allow-origin "https://domain.com"
referrer-policy "no-referrer"
content-security-policy "upgrade-insecure-requests; default-src 'none'; base-uri 'none'; connect-src 'self'; font-src 'self'; form-action 'self'; frame-ancestors 'none'; img-src ‘self’; media-src 'self'; object-src ‘none’ ; script-src 'self'; script-src-attr 'none'; style-src 'self'"
permissions-policy "geolocation=(), …"
cross-origin-embedder-policy: "require-corp"
cross-origin-opener-policy "same-origin"
cross-origin-resource-policy "cross-origin"
```

In addition be sure to remove _Server_ and _X-Powered-By_ headers.

> [!NOTE]
> Never use _X-XSS-Protection_, it is depracated and can create XSS vulnerabilities in otherwise safe websites. _X-Frame-Options_ is depracated and replaced by _frame-ancestors 'none'_. Always start with _default-src 'none'_, avoid _unsafe-inline_ and _unsafe-eval_. Use hashes or nonces for inline scripts/styles.

## **HTML DOM sanitization**

### **Links**

Always use _rel="noreferrer noopener"_ to prevent the referrer header from being sent to the new page.

### **POST vs GET**

Never trust user inputs, validate and sanitize all data. Prefer POST requests instead of GET requests and sanitize/encode user form data with a strong regex and _application/json_.

```js
try {
  if (!yourData ||
    // regex, sanitize, etc.
  ) return
  const data = JSON.stringify({ yourData })
  const res = await fetch('api/fetch/', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: data,
  })
  if (!res.ok) {
    //
    return
  }
  //
} catch {
  // 
}
```

### **e.innerHTML**

Never use _innerHTML_; use _innerText_ or _textContent_ instead. You can also create your element with _document.createElement()_.

### **eval() and new Function()**

Never use these JavaScript function. Executing JavaScript from a string is an enormous security risk. It is far too easy for a bad actor to run arbitrary code when you use _eval()_.

### **DOMPurify**

DOMPurify sanitizes HTML and prevents XSS attacks. You can feed DOMPurify with string full of dirty HTML and it will return a string (unless configured otherwise) with clean HTML. DOMPurify will strip out everything that contains dangerous HTML and thereby prevent XSS attacks and other nastiness.

```js
import DOMPurify from 'dompurify'

const PURIFY_CONFIG = {
  SANITIZE_NAMED_PROPS: true,
  ALLOW_DATA_ATTR: false,
  ALLOWED_URI_REGEXP: /^(https?|mailto|tel):/i,
  FORBID_TAGS: ['dialog', 'footer', 'form', 'header', 'iframe', 'main', 'nav', 'script', 'style'],
  FORBID_ATTR: ['style', 'class', 'onclick', 'onload', 'onerror']
}

const clean = DOMPurify.sanitize(dirty, PURIFY_CONFIG)
```

## **Analysis tools**

[**Mozilla Observatory**](https://developer.mozilla.org/en-US/observatory)

[**SSLLabs**](https://www.ssllabs.com/ssltest/)

[**Cryptcheck**](https://cryptcheck.fr/)

[**W3C Validator**](https://validator.w3.org/)

## **Sources and resources**

<https://developer.mozilla.org/en-US/>

<https://www.cnil.fr/fr/securiser-vos-sites-web-vos-applications-et-vos-serveurs>

<https://cyber.gouv.fr/publications/recommandations-de-securite-relatives-tls>

<https://owasp.org/www-project-top-ten/>

<https://cheatsheetseries.owasp.org/>

<https://www.cert.ssi.gouv.fr/>

<https://phpdelusions.net/>

<https://expressjs.com/en/advanced/best-practice-security.html>

<https://www.digitalocean.com/community/tutorials>

<https://thehackernews.com/>

<https://portswigger.net/daily-swig/zero-day>
