# NodeJS-Exploitation
Basic NodeJS Exploitation for data exfiltration and reverse shells

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
http://127.0.0.1:8080
```
Attacker

Pass it user input
```
http://127.0.0.1:8080/?q='Test'
```

Exfiltrate the /etc/passwd file of the remote system

```
http://127.0.0.1:8080/?q='Test'

```






