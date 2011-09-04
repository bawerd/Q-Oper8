# Q-Oper8

Simple multi-process manager for Node.js

Rob Tweed <rtweed@mgateway.com>  
30 August 2011, M/Gateway Developments Ltd [http://www.mgateway.com](http://www.mgateway.com)  

Twitter: @rtweed

## Installing Q-Oper8

       npm install qoper8

Note: *Q-Oper8* requires Node.js version 0.5.x or later, as it makes use of its child Node process
capability
	   
##  What is Q-Oper8?

*Q-Oper8* is a simple module for enabling and managing a scalable but high-performance multi-process 
environment in Node.js.  The primary incentive for writing it was to create a hybrid environment where 
the many benefits of synchronous coding could be acheived without the risk of blocking the main server process.

It consists of four components:

- a master Node server process that is 100% asynchronous and non-blocking
- a pool of child Node processes that **only ever handle a single action/request at a time**
- a queue of pending actions/requests
- a queue processor that attempts to allocate requests/actions on the queue to available child Node processes

The child processes persist throughout the lifetime of the master Node server process, so there is no setup/teardown 
overhead or delay when handling requests/actions: the child processes that are flagged as being in the 
*available process pool* are instantaneously available for use by the master process.

It's the fact that the Child Node Processes only ever handle a single request at a time that makes it 
possible for them to support synchronous coding, since they don't need to worry about blocking anyone else.

Note: *Q-Oper8* is completely event-driven with no polling overheads.

You have complete control over the behaviour and configuration of *Q-Oper8*.  You can:

- determine the number of child Node processes that constitute the worker pool
- write the child process logic that you require for handling your actions/requests
- define a handler that runs on the master server to process the completed responses sent back from the child processes, eg 
  to return the response as a web page to the originating client.

Clearly, the larger the pool of Node.js processes, the less likely it is that the request/action queue will build up.  On the 
other hand, each child Node process uses about 10Mb memory according to the Node.js documentation.  Additionally, the quicker 
your child process can handle a request, the sooner it will become available again to the pool to handle a queued request/action.

##  Benefits of the Q-Oper8 module

The *Q-Oper8* module addresses many of the key potential drawbacks of Node.js, including:

- allowing safe use of synchronous logic with Node.js, avoiding the need for cumbersome, non-intuitive and difficult to maintain 
  nested callback structures.  Synchronous logic won't block the main server process
- allowing the safe use of in-process synchronous database APIs, as exemplified by the [Globals](http://glogalsdb.org) database
- providing protection to your main Node server process by isolating it from problems that might occur when handling particular 
  requests/actions.
- distributing load across multiple Node processes, allowing Node to exploit multiple-core CPUs.
- providing a highly scalable architecture that can be easily tailored to suit your traffic and processing demands.

##  Using the Q-Oper8 module

Node.js 0.5.x must be installed

The /examples directory contains a simple worked example of a master server process and a child process to handle web requests.

The following is a simple example of how to use the *Q-Oper8* module:

      var qoper8 = require('qoper8');
	  
      qoper8.start('', function() {
        console.log("Q-Oper8 started!!!");
		// start processing!
      });

An action is defined as a standard Javascript object.  Its structure and content is up to you, but note that the object that 
is passed to a child Node process cannot contain functions.  

You simply add your action to thread's queue and let it do the rest, eg:

       var requestObj = {action: {x:1,y:2}, response: response, otherStuff: 'etc...'};
       qoper8.addToQueue(requestObj, responseHandler);

      
The only part of the requestObj object that is sent to and handled by your child process is the *action* property.  
You can add any other properties to the requestObj object: these are retained within the master Node process 
for use by *responseHandler*: the master Node process response handler that you must also define (see later).

So, in the example above, if we're using the scheduler with Node's HTTP server module, we're adding the response object to the 
requestObj object so that the master Node process has the correct handle to allow it to ultimately return the response to the correct client.

##  Startup parameters

The parameters that you can specify for the *Q-Oper8* *start()* function are as follows:

- poolSize = the number of Node child processes to fire up (deafult = 5)
- childProcessPath = the filepath of the Node child process Javascript file (default = __dirname + '/qoper8ChildProcess.js')
- monitorInterval = no of milliseconds delay between displaying process usage in console (default = 30000)
- trace = true if you want to get a detailed activity trace to the Node.js console (default = true)
- silentStart = true if you don't want any message to the console when *Q-Oper8* starts (default = false)

For example:

      var qoper8 = require('qoper8');
	  
	  var params = {poolSize: 20, childProcessPath: '/home/user/node/myChildProc.js', trace: false};
      qoper8.start(params, function() {
        console.log("Q-Oper8 started!!!");
		// start processing!
      });

##  Defining a child Node process

The child process should be designed to handle any instance of your action/requests that get sent to it by the master
Node process.  Note that a child Node process will only handle one single action/request at a time, so you are at liberty to 
use as much synchronous coding as you like, since there will be no other users to block.

Here's a simple example:

       var childProcess = require('qoper8').childProcess;
       
	   var actionMethod = function(action) {
         console.log("Action method: Process " + process.pid + ": action = " + JSON.stringify(action));
         var result = "method completed for " + process.pid + " at " + new Date().toLocaleTimeString();
         return result;
       };

       childProcess.handler(actionMethod);

So you first define your *actionMethod* which will process the contents of the action object that you originally placed 
on the queue.  This method can do anything you like, and must return a value or object.  This returnValue will be automatically sent back to 
the master Node process which will look after what is done with it, eg sending it back to a user as a web page.

Then just add the last line exactly as shown above.  That's it!  *Q-Oper8* will do the rest.

## Defining the master Node Results Handler method

You define this in the master Node process.  Here's an example that will package up the resonses from the child Processes
as web pages:

      var responseHandler = function(requestObj, results) {
        //console.log("This is the response handler: ");
        //console.log("** action: " + JSON.stringify(requestObj.action));
        //console.log("results = " + JSON.stringify(results));

        var response = requestObj.response;
        var html = "<html>";
        html = html + "<head><title>Q-Oper8 action response</title></head>";
        html = html + "<body>";
        html = html + "<p>Action was processed !</p><p>Results: " + results + "</p>";
        html = html + "</body>";
        html = html + "</html>";

        response.writeHead(200, {"Content-Type": "text/html"});  
        response.write(html);  
        response.end();  
      };

- requestObj will be picked up automatically by *Q-Oper8* and is the original requestObj you placed on the queue
- results is the returnValue you returned from your childProcesses.

You add a reference to this handler whenever you add a request/action to the *Q-Oper8* queue, eg:

       qoper8.addToQueue(requestObj, responseHandler);

## That's it!

You now have the best of all worlds: a non-blocked scalable Node server process, with a pool of child Node processes in which you 
can use synchronous logic without any concern about blocking other users.  You can also make use of your multi-core processor into 
the bargain!

## Running the Example

- Find the simple worked example in the */examples* directory of this repository.

- Copy *qoper8ChildProcess.js* to the same directory/path used by npm when you installed the *Q-Oper8* module, 
   eg *~/node/node_modules/qoper8/lib* (Note: you can change the name and location of the childProcess file by 
   using the *childProcessPath* startup parameter for *Q-Oper8*)

- Copy *webQOper8.js* to the directory/path where you normally run your Node.js applications, eg *~/node*

- Start the example: **node webQOper8.js**

You'll now have a web server running on port 8080 (edit *webQOper8.js* if you want to use a different port)

Now start a browser and point it at the Node application's IP address and port.  Make sure you use a URL that includes */test/*, eg:

       http://192.168.1.100:8080/test/it
	   
You should get a response that is something like:

       Action was processed!
	   
	   Results: method completed for 10372 at 09:40:07
	   
You should also have seen a flurry of activity in the Node.js console, because the *trace* flag in *Q-Oper8* has been set to *true*.

The key lines that made it all burst into life are in *webQOper8.js*:

       // *********Example use of threaded action ******** 

       if (uri.indexOf('/test/') !== -1) {
         var action = {query: urlObj.query};
         var requestObj = {action: action, request: request, response: response, urlObj: urlObj};
         qoper8.addToQueue(requestObj, handler);
       }

       // **************************************************

ie the request was added to the *Q-Oper8* queue when you sent the URL.  The request was then processed by the *actionMethod* in 
one of the running instances of *qoper8ChildProcess.js* and the HTML response was generated by the *handler* function in *webQOper8.js*

Now try it out with your own applications!
	   
Enjoy!
	   
## License

Feel free to download, use and redistribute without restriction.  All I really ask for is acknowledgement of your use or 
modification of this module, and respect for its copyright.

Copyright (c) 2011 M/Gateway Developments Ltd,
Reigate, Surrey UK.
All rights reserved.

http://www.mgateway.com
Email: rtweed@mgateway.com

This program is free software: you can redistribute it and/or modify it under the terms of the GNU Affero General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License along with this program.  If not, see <http://www.gnu.org/licenses/>.

