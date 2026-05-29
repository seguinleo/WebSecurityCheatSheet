# WebSecurityCheatSheet

## 📖 Table of contents

- [**HTTPS (i.e. Apache2)**](#https-ie-apache2)
  - [**Let's Encrypt**](#lets-encrypt)
  - [**Redirect all HTTP traffic to HTTPS**](#redirect-all-http-traffic-to-https)
  - [**Transport Layer Security (TLS)**](#transport-layer-security-tls)
  - [**HTTP Strict Transport Security (HSTS)**](#http-strict-transport-security-hsts)
- [**Apache2**](#apache2)
  - [**Enable HTTP2**](#enable-http2)
  - [**Enable mod\_security**](#enable-mod_security)
  - [**Enable Gzip**](#enable-gzip-or-brotli)
  - [**Hide server signature**](#hide-server-signature)
  - [**Restrict access to files**](#restrict-access-to-files)
- [**Nginx**](#nginx-recommended)
- [**HTTP Headers**](#http-headers)
- [**Authorization**](#authorization)
- [**Database**](#database)
- [**PHP**](#php)
  - [**PHP-FPM**](#php-fpm)
  - [**PHP PDO**](#php-pdo)
  - [**php.ini**](#phpini)
- [**Node.js/npm**](#nodejsnpm)
- [**Express**](#express)
  - [**Server**](#server)
  - [**Express session**](#express-session)
- [**User registration**](#user-registration)
- [**User login**](#user-login)
- [**CSRF Token**](#csrf-token)
- [**Cookies**](#cookies)
- [**File upload**](#file-upload)
- [**Docker**](#docker)
- [**Ubuntu VPS**](#ubuntu-vps)
- [**HTML DOM sanitization**](#html-dom-sanitization)
  - [**Zod**](#zod)
  - [**Links**](#links)
  - [**POST vs GET**](#post-vs-get)
  - [**e.innerHTML**](#einnerhtml)
  - [**eval() and new Function()**](#eval-and-new-function)
  - [**DOMPurify**](#dompurify)
- [**Analysis tools**](#analysis-tools)
- [**Sources and resources**](#sources-and-resources)

**_This document is a concise guide that aims to list the main web vulnerabilities, particularly JavaScript, and some solutions. This guide should be supplemented with quality, up-to-date documentation._**

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

### **Redirect www to non-www**

```apache
RewriteCond %{HTTPS} off [OR]
RewriteCond %{HTTP_HOST} ^www\.(.*)$ [NC]
RewriteRule ^ https://example.com%{REQUEST_URI} [L,R=301]
```

### **Transport Layer Security (TLS)**

General purpose web applications should default to TLS 1.3 (support TLS 1.2 if necessary) with all other protocols disabled. Only enable TLS 1.2 and 1.3. Go to _/etc/apache2/conf-available/ssl.conf_ and write:

```apache
SSLProtocol TLSv1.2 TLSv1.3
```

### **HTTP Strict Transport Security (HSTS)**

HTTP Strict Transport Security (HSTS) is a mechanism for websites to instruct web browsers that the site should only be accessed over HTTPS. This mechanism works by sites sending a Strict-Transport-Security HTTP response header containing the site's policy. Write in _/etc/apache2/apache2.conf_:

```apache
Header set strict-transport-security "max-age=31536000; includesubdomains; preload"
```

Reload Apache and submit your website to <https://hstspreload.org/>

> [!IMPORTANT]
> Only enable preload if you're 100% HTTPS everywhere (including subdomains)

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

### **Enable Gzip (or Brotli)**

``a2enmod deflate``

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

## **Nginx (recommended)**

A nginx template following the same Apache2 config above:

```nginx
server {
  listen 80;
  server_name your-website.com www.your-website.com;

  return 301 https://$host$request_uri;
}

# Redirect www to non www
server {
  listen 443 ssl http2;
  server_name www.your-website.com;

  ssl_certificate /etc/ssl/cert.pem;
  ssl_certificate_key /etc/ssl/key.pem;

  return 301 https://your-website.com$request_uri;
}

server {
  listen 443 ssl http2;
  server_name your-website.com www.your-website.com;

  ssl_certificate /etc/ssl/cert.pem;
  ssl_certificate_key /etc/ssl/key.pem;

  ssl_protocols TLSv1.2 TLSv1.3;

  root /usr/share/nginx/html;
  index index.html;

  server_tokens off;

  gzip on;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  location / {
    try_files $uri $uri/ /index.html;

    limit_except GET POST {
      deny all;
    }
  }
}
```

## **HTTP Headers**

Recommended headers for Apache2 and nginx:

```apache
x-content-type-options: "nosniff"
access-control-allow-origin "https://domain.com"
referrer-policy "no-referrer"
content-security-policy "upgrade-insecure-requests; default-src 'self'; base-uri 'none'; connect-src 'self'; font-src 'self'; form-action 'self'; frame-ancestors 'none'; img-src 'self'; media-src 'self'; object-src 'none' ; script-src 'self'; script-src-attr 'none'; style-src 'self'"
permissions-policy "geolocation=(), …"
cross-origin-embedder-policy: "require-corp"
cross-origin-opener-policy "same-origin"
cross-origin-resource-policy "cross-origin"
```

In addition be sure to remove _Server_ and _X-Powered-By_ headers.

> [!NOTE]
> Never use _X-XSS-Protection_, it is depracated and can create XSS vulnerabilities in otherwise safe websites. _X-Frame-Options_ is replaced by _frame-ancestors 'none'_. Avoid as much as possible _unsafe-inline_ and _unsafe-eval_. Use hashes or nonces for inline scripts/styles.

## **Authorization**

- Deny by default
- Enforce least privileges
- Validate all permissions
- Validate files access
- Sanitize files upload
- Require user password for sensitive actions

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
- For NoSQL databases, like MongoDB, use a typed model to prevent some injections
- Avoid _$accumulator_, _$function_, _$where_ in MongoDB
- Store keys in a secure vault like AWS Secrets Manager, Azure KeyVault or Hashicorp

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

## **Node.js/npm**

- Always keep all npm dependencies up to date, use _npm audit_
- Limit the use of dependencies
- Use _npm doctor_ to ensure that your npm installation has what it needs to manage your JavaScript packages
- Use eslint to write quality code

## **Express**

### **Server**

[RECOMMENDED] Secure Express server (behind reverse proxy):

```js
import express from 'express'
import helmet from 'helmet'

const app = express()

// If behind a reverse proxy like Nginx
app.set('trust proxy', 1)

// If requests are application/json
app.use(express.json({ limit: '50kb' }))

// Security headers
app.use(helmet())
app.disable('x-powered-by')

const PORT = process.env.PORT || 3000

app.listen(
  PORT,
  '127.0.0.1', // bind only to localhost if possible
  () => {
  console.log(`Server is running on port ${PORT}`)
})
```

Or, secure Express server with HTTPS (WITHOUT reverse proxy):

```js
import express from 'express'
import helmet from 'helmet'
import https from 'https'
import fs from 'fs'

const app = express()

app.use(express.json({ limit: '50kb' }))
app.use(helmet())
app.disable('x-powered-by')

const PORT = process.env.PORT || 3000

https.createServer({
  key: fs.readFileSync(process.env.SSL_KEY),
  cert: fs.readFileSync(process.env.SSL_CERT),
}, app).listen(PORT, () => {
  console.log(`HTTPS server running on port ${PORT}`)
})
```

If you want CORS requests:

```js
import cors from 'cors'

const corsOptions = {
  origin: process.env.YOUR_URL,
  credentials: true,
}
app.use(cors(corsOptions))
```

Secure all routes with _express-rate-limit_:

```js
const router = express.Router()

const limiter = rateLimit({
  windowMs: 5 * 60 * 1000,
  max: 100,
  message: 'Too many requests, please try again later.',
})

router.use(limiter)
```

### **Express session**

If you are using _express-session_, secure it and store in _Redis_:

```js
import express from 'express'
import session from 'express-session'
import { RedisStore } from 'connect-redis'
import { createClient } from 'redis'

const app = express()

const PORT = process.env.PORT || 3000

const redisClient = createClient({
  url: process.env.REDIS_URL,
})

try {
  await redisClient.connect()
  console.log('Redis client connected')
} catch (err) {
  console.error(err)
  process.exit(1)
}

const redisStore = new RedisStore({
  client: redisClient,
  prefix: 'your_app:',
  ttl: 3600,
})

app.use(
  session({
    store: redisStore,
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
      httpOnly: true,
      secure: true,
      sameSite: 'Strict',
    },
  })
)
```

## **User registration**

A secure Express user registration using _Argon2id_ and _zod_:

```js
const createAccountSchema = z.object({
  nameCreate: z
    .string()
    .min(3)
    .max(30)
    .regex(/^[\p{L} -]+$/u),
  psswdCreate: z.string().min(10).max(64)
})

router.post('/create-account', async (req, res) => {
  const parsed = createAccountSchema.safeParse(req.body)
  if (!parsed.success) {
    return res.status(400).send('Account creation failed')
  }

  const { nameCreate, psswdCreate } = parsed.data

  const id = crypto.randomBytes(12).toString('hex')
  const psswdCreateHash = await argon2.hash(psswdCreate)

  try {
    await pool.execute(
      "INSERT INTO users (id, name, psswd) VALUES (?, ?, ?)",
      [id, nameCreate, psswdCreateHash]
    )
    return res.status(200).send('Account created successfully')
  } catch {
    return res.status(400).send('Account creation failed')
  }
})
```

## **User login**

A secure Express login system using rate limiting and _passport.js_:

```js
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

    req.session.regenerate((err) => {
      if (err) return res.status(401).send('Wrong username or password.')

      // create session here
    })

    return res.status(200).send('Logged in!')
  } catch {
    return res.status(401).send('Wrong username or password.')
  }
})
```

**Use MFA (2FA) for sensible apps !**

## **CSRF Token**

SameSite cookie is good as defense in depth, but doesn't prevent all possible CSRF attacks. Use CSRF Token or Double Submit Cookie. [Read OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)

I recommend [csrf-csrf](https://github.com/Psifi-Solutions/csrf-csrf) for Express.

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
> Since using SameSite with the **Strict** attribute is relatively safe, it is also recommended to use a CSRF token for sensible apps

## **File upload**

Always the extension AND the actual MIME type AND size before saving it:

```js
import multer from 'multer'
import path from 'path'
import fs from 'fs'
import { fileTypeFromBuffer } from 'file-type'

const uploadDir = '/var/uploads'

if (!fs.existsSync(uploadDir)) {
  fs.mkdirSync(uploadDir, { recursive: true })
}
const ALLOWED_TYPES = [
  {
    mime: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    ext: 'xlsx'
  }
]

const storage = multer.memoryStorage()

export const upload = multer({
  storage,
  limits: { fileSize: 512000 }
})

export const handleUpload = async (req, res, next) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' })
    }

    const buffer = req.file.buffer

    const fileType = await fileTypeFromBuffer(buffer)

    if (!fileType) {
      return res.status(400).json({ error: 'Unknown file type' })
    }

    const allowed = ALLOWED_TYPES.find(
      t => t.mime === fileType.mime && t.ext === fileType.ext
    )

    if (!allowed) {
      return res.status(400).json({ error: 'Invalid file type' })
    }

    const originalName = req.file.originalname
    const baseName = path.parse(originalName).name
    const safeName = baseName.replace(/[^a-z0-9_\-]/gi, '_')
    const finalName = `${safeName}.${fileType.ext}`
    const finalPath = path.join(uploadDir, finalName)

    await fs.promises.writeFile(finalPath, buffer)

    next()
  } catch (err) {
    next(err)
  }
}
```

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
- Use Fail2Ban to prevent DoS and Bruteforce attacks, disable SSH root login in _sshd_config_

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

## **HTML DOM sanitization**

### **Zod**
Zod is a great tool to sanitize inputs:

```js
import * as z from 'zod'

const User = z.object({
  name: z.string().min(3).max(64),
})
```

### **Links**

Always use _rel="noreferrer noopener"_ to prevent the referrer header from being sent to the new page.

### **POST vs GET**

Never trust user inputs, validate and sanitize all data. Use GET for idempotent requests, POST for state changes. Sanitize and valid data with a strong regex and use _application/json_ instaed of _application/x-www-form-urlencoded_:

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

Never use _innerHTML_ without DOMPurify; use _innerText_ or _textContent_ instead. You can also create your element with _document.createElement()_.

### **eval() and new Function()**

Never use these JavaScript function. Executing JavaScript from a string is an enormous security risk. It is far too easy for a bad actor to run arbitrary code when you use _eval()_.

### **DOMPurify**

DOMPurify sanitizes HTML and prevents XSS attacks. You can feed DOMPurify with string full of dirty HTML and it will return a string (unless configured otherwise) with clean HTML. DOMPurify will strip out everything that contains dangerous HTML and thereby prevent XSS attacks and other nastiness.

```js
import DOMPurify from 'dompurify'

const PURIFY_CONFIG = {
  SANITIZE_NAMED_PROPS: true,
  ALLOW_DATA_ATTR: false,
  ALLOWED_URI_REGEXP: /^(https?|mailto|tel):/i
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
