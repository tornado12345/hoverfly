[![Circle CI][CircleCI-Image]][CircleCI-Url]
[![ReportCard][ReportCard-Image]][ReportCard-Url]

![Hoverfly](static/images/hf-logo-std-r-transparent-medium.png)
## Dependencies without the sting

[Hoverfly](http://hoverfly.io) is a lightweight, open source [service virtualization](https://en.wikipedia.org/wiki/Service_virtualization) tool.
Using Hoverfly, you can virtualize your application dependencies to create a portable, self-contained development or test sandbox.

Hoverfly is a proxy written in [Go](https://github.com/golang/go). It allows you to create "simulated" dependencies by:

* Capturing HTTP(s) traffic between your application and external services
* Building your own "synthetic" services from scratch

Another powerful feature: middleware modules, where users
can introduce their own custom logic. **Middleware modules can be written in any language**.

You can use middleware to simulate network latency, failure, rate limits or malformed responses, or generate responses on the fly.

More information about Hoverfly and how to use it:
* [Creating fast versions of slow dependencies](http://www.specto.io/blog/speeding-up-your-slow-dependencies.html)
* [Modifying traffic on the fly](http://www.specto.io/blog/service-virtualization-is-so-last-year.html)
* [Mocking APIs for development and testing](http://www.specto.io/blog/api-mocking-for-dev-and-test-part-1.html)
* [Virtualizing the Meetup API](http://www.specto.io/blog/hoverfly-meetup-api.html)
* [Using Hoverfly to build Spring Boot microservices alongside a Java monolith](http://www.specto.io/blog/using-api-simulation-to-build-microservices.html)
* [Easy API simulation with the Hoverfly JUnit rule](https://specto.io/blog/hoverfly-junit-api-simulation.html)

## Installation

### Pre-built binary

Pre-built Hoverfly binaries are available [here](https://github.com/SpectoLabs/hoverfly/releases/).  

#### OSX, Linux & *BSD

Download a binary, set the correct permissions:

    chmod +x hoverfly*

And run it:

    ./hoverfly*

#### Windows

Simply download a .exe file and run it.

### Build it yourself  

To set up your Go environment - look [here](https://golang.org/doc/code.html).

You must have Go > 1.6 installed.

    mkdir -p "$GOPATH/src/github.com/SpectoLabs/"
    git clone https://github.com/SpectoLabs/hoverfly.git "$GOPATH/src/github.com/SpectoLabs/hoverfly"
    cd "$GOPATH/src/github.com/SpectoLabs/hoverfly"
    make build

And to run hoverfly:

    ./cmd/hoverfly/hoverfly


## Admin UI

The Hoverfly admin UI is available at [http://localhost:8888/](http://localhost:8888/). It uses the API
(as described below) to change state. It also allows you to wipe the captured requests/responses and shows the number
of captured records. For other functions, such as export/import, you can use the API directly.

## Hoverfly is a proxy

Configuring your application to use Hoverfly is simple. All you have to do is set the HTTP_PROXY environment variable:

     export HTTP_PROXY=http://localhost:8500/

## Destination configuration

You can specify which site to capture or virtualize with a regular expression (by default, Hoverfly processes everything):

    ./hoverfly -destination="."

Or you can also provide '-dest' flags to identify regexp patterns of multiple hosts that should be virtualized:

    ./hoverfly -dest www.myservice.org -dest authentication.service.org -dest some-other-host.com

Hoverfly will process only those requests that have a matching destination. Other requests will be passed through. This allows you
to start by virtualizing just a view services, adding more later.  

## Modes (Virtualize / Capture / Synthesize / Modify)

Hoverfly has different operating modes. Each mode changes the behavior of the proxy. Based on the selected mode, Hoverfly can
either capture the requests and responses, look for them in the cache, or send them directly to the middleware and
respond with a payload that is generated by the middleware (more on middleware below).

### Virtualize

By default, the proxy starts in virtualize mode. You can apply middleware to each response.

### Capture

When capture mode is active, Hoverfly acts as a "man-in-the-middle". It makes requests on behalf of a client and records
the responses. The response is then sent back to the original client.

To switch to capture mode, you can add the "--capture" flag during startup:

    ./hoverfly --capture

Or you can select "Capture" using the admin UI at [http://localhost:8888/](http://localhost:8888/)

Or you can use API call to change proxy state while running (see API below).

Do a curl request with proxy details:

    curl http://mirage.readthedocs.org --proxy http://localhost:8500/

###  Synthesize

Hoverfly can create responses to requests on the fly. Synthesize mode intercepts requests (it also respects the --destination flag)
and applies the supplied middleware (the user is required to supply --middleware flag, you can read more about it below). The middleware
is expected to populate response payload, so Hoverfly can then reconstruct it and return it to the client. An example of a synthetic
service can be found in _this_repo/examples/middleware/synthetic_service/synthetic.py_. You can test it by running:

    ./hoverfly --synthesize --middleware "../../examples/middleware/synthetic_service/synthetic.py"

### Modify

Modify mode applies the selected middleware (the user is required to supply the --middleware flag - you can read more about this below)
to outgoing requests and incoming responses (before they are returned to the client). It
allows users to use Hoverfly without storing any data about requests. It can be used when it's difficult to - or there is little point
in - describing a service, and you only need to change certain parts of the requests or responses (eg: adding authentication headers through with middleware).
The example below changes the destination host to "mirage.readthedocs.org" and sets the method to "GET":

    ./hoverfly --modify --middleware "../../examples/middleware/modify_request/modify_request.py

## HTTPS capture 

Hoverfly can generate public and private keys:

    ./hoverfly -generate-ca-cert

'cert.pem' and 'key.pem' should appear in your current directory. Add cert.pem to your trusted certificates. Next time when you run Hoverfly -
specify this key and certificate:

    ./hoverfly -cert cert.pem -key key.pem

You can also use default certificate that is inside Hoverfly. Add cert.pem from project root to your trusted certificates or 
turn off verification. With curl you can make insecure requests with -k:

    curl https://www.bbc.co.uk --proxy http://localhost:8500 -k


### Turning off certificate verification (Hoverfly will be happy with untrusted certificates)

You can specify Hoverfly to ignore untrusted certificates when it's capturing or modifying traffic through command line flag:

    ./hoverfly -tls-verification=false
    
Or you can also do it through environment variable 'HoverflyTlsVerification' like this:

    export HoverflyTlsVerification=false


## API

### Authentication

Hoverfly uses a combination of basic auth and JWT (JSON Web Tokens) to authenticate users.

### Authentication (enabled by default)

To disable authentication for you can pass '-no-auth' flag during startup:

    ./hoverfly -no-auth
    
or supply environment variable:

    export HoverflyAuthDisabled=true
    
If environment variable __or__ flag is given to disabled authentication - it will be disabled. If you disable authentication -
you can use any username/password pair to authenticate to admin user interface.

Export Hoverfly secret:

    export HoverflySecret=VeryVerySecret
    

If you skip this step - a new random secret will be generated every single time when you launch Hoverfly. This can be useful
if you are deploying it in the cloud but it can also be annoying if you are working with Hoverfly where it is constantly restarted.

You can also specify token expiration time (defaults to 1 day) in seconds:

    export HoverflyTokenExpiration=3600

### Initial super user
You can define initial super user through environment variables (useful if you are using Docker or AMI) by exporting these
environment variables:

    export HoverflyAdmin="hf" 
    export HoverflyAdminPass="hf"
    
Another option is to manually add user during startup. If no user is present in Hoverfly database and authentication 
is enabled - you will be asked to enter username and password for the initial admin level user so you can use the API. 

### Adding users

Then, add your first admin user:

    ./hoverfly -v -add -username hfadmin -password hfadminpass
     
You can also create non-admin users by supplying 'admin' flag as follows:

    ./hoverfly -v -add -username hfadmin -password hfadminpass -admin false

Getting token:

    curl -H "Content-Type application/json" -X POST -d '{"Username": "hoverfly", "Password": "testing"}' http://localhost:8888/api/token-auth

Using token:

    curl -H "Authorization: Bearer eyJhbGciOiJSUzUxMiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE0NTYxNTY3ODMsImlhdCI6MTQ1NTg5NzU4Mywic3ViIjoiIn0.Iu_xBKzBWlrO70kDAo5hE4lXydu3bQxDZKriYJ4exg3FfZXCqgYH9zm7SVKailIib9ESn_T4zU-2UtFT5iYhw_fzhnXtQoBn5HIhGfUb7mkx0tZh1TJBkLCv6y5ViPw5waAnFBRcygh9OdeiEqnJgzHKrxsR87EellXSdMn2M8wVIhjIhS3KiDjUwuqQl-ClBDaQGlsLZ7eC9OHrJIQXJLqW7LSwrkV3rstCZkTKrEZCdq6F4uAK0mgagTFmuyaBHDEccaivkgYDcaBb7n-Vmyh-jUnDOnwtFnrOv_myXlqqkvtezfm06MBl4PzZE6ZtEA5XADdobLfVarbvB9tFbA" http://localhost:8888/api/records


### Usage

You can access the administrator API under the default hostname of 'localhost' and port '8888':

* Recorded requests: GET [http://localhost:8888/api/records](http://localhost:8888/api/records):


    curl http://localhost:8888/api/records

* Wipe cache: DELETE http://localhost:8888/api/records:


    curl -X DELETE http://localhost:8888/api/records

* Get current proxy state: GET [http://localhost:8888/api/state](http://localhost:8888/api/state)
* Set proxy state: POST http://localhost:8888/api/state
   + body to start virtualizing: {"mode":"virtualize"}
   + body to start capturing: {"mode":"capture"}


    curl -H "Content-Type application/json" -X POST -d '{"mode":"capture"}' http://localhost:8888/api/state


* Update proxy destination: POST http://localhost:8888/api/state


    curl -H "Content-Type application/json" -X POST -d '{"destination": "service-hostname-here"}' http://localhost:8888/api/state

* Exporting recorded requests to a file:


    curl http://localhost:8888/api/records > requests.json

* Importing requests from file:


    curl --data "@/path/to/requests.json" http://localhost:8888/api/records

#### Metadata

Hoverfly metadata API provides easy to use key/value storage to make management of multiple Hoverflies easier. Currently it automatically adds metadata with imported sources
if it was done using launching flags ("-import http://mystorehostname/service.json").

You can also set any key/values yourself. Let's give our Hoverfly a name! It would look something like this:

    curl -H "Content-Type application/json" -X PUT -d '{"key":"name", "value": "Mad Max"}' http://localhost:8888/api/metadata

and then add some description to it:

    curl -H "Content-Type application/json" -X PUT -d '{"key":"description", "value": "Simulates keystone service, use user XXXX and password YYYYY to login"}' http://localhost:8888/api/metadata

then, to get your Hoverfly's info - use GET request:

    curl http://localhost:8888/api/metadata

result should be something like this:
```javascript
{
	"data": {
		"description": "Simulates keystone service, use user XXXX and password YYYYY to login",
		"name": "Peters Hoverfly"
	}
}
```

if you import some resources during launch:

    ./hoverfly -import ../../resources/keystone_service.json -import ../../resources/nova_service.json

our metadata will look like:

```javascript
{
	"data": {
		"description": "Simulates keystone service, use user XXXX and password YYYYY to login",
		"import_1": "../../resources/keystone_service.json",
		"import_2": "../../resources/nova_service.json",
		"name": "Peters Hoverfly"
	}
}
```


Currently available commands:
* Set metadata: PUT http://localhost:8888/api/metadata ( __curl -H "Content-Type application/json" -X PUT -d '{"key":"my_key", "value": "my_value"}' http://localhost:8888/api/metadata__ )
* Get all metadata: GET [http://localhost:8888/api/metadata](http://localhost:8888/api/metadata) ( __curl http://localhost:8888/api/metadata )
* Delete all metadata: DELETE http://localhost:8888/api/metadata ( __curl -X DELETE http://localhost:8888/api/metadata )

## Importing data on startup:

Hoverfly can import data on startup from given file or url:

    ./hoverfly -import my_service.json

or:

    ./hoverfly -import http://mypage.com/service_x.json


## Middleware

Hoverfly supports external middleware modules. You can write them in __any language you want__ !.
These middleware modules are expected to take standard input (stdin) and should output a structured JSON string to stdout.
Payload example:

```javascript
{
	"response": {
		"status": 200,
		"body": "body here",
		"headers": {
			"Content-Type": ["text/html"],
			"Date": ["Tue, 01 Dec 2015 16:49:08 GMT"],
		}
	},
	"request": {
		"path": "/",
		"method": "GET",
		"destination": "1stalphaomega.readthedocs.org",
		"query": ""
	},
	"id": "5d4f6b1d9f7c2407f78e4d4e211ec769"
}
```
Middleware is executed only when request is matched so for fully dynamic responses where you are
generating response on the fly you can use ""--synthesize" flag.

In order to use your middleware, just add path to executable:

    ./hoverfly --middleware "../../examples/middleware/modify_response/modify_response.py"

You should be treating Hoverfly middleware as commands, this enables a bit more complex execution:

    .hoverfly --middleware "go run ../../examples/middleware/go_example/change_to_custom_404.go"

In this example Hoverfly is also compiling Go middleware on the fly. However this increases time taken to execute middleware
(and therefore to respond) so it's better to precompile middleware binaries before using them.

#### Python middleware example

Basic example of a Python module to change response body, set different status code and add 2 second delay:

```python
#!/usr/bin/env python
import sys
import logging
import json
from time import sleep


logging.basicConfig(filename='middleware.log', level=logging.DEBUG)
logging.debug('Middleware is called')


def main():
    data = sys.stdin.readlines()
    # this is a json string in one line so we are interested in that one line
    payload = data[0]
    logging.debug(payload)

    payload_dict = json.loads(payload)

    payload_dict['response']['status'] = 201
    payload_dict['response']['body'] = "body was replaced by middleware"

    # now let' sleep for 2 seconds
    sleep(2)

    # returning new payload
    print(json.dumps(payload_dict))

if __name__ == "__main__":
    main()

```

Save this file with python extension, _chmod +x_ it and run hoverfly:

    ./hoverfly --middleware "./this_file.py"

#### JavaScript middleware example

You can also execute JavaScript middleware using Node. Make sure that you can execute Node, you can brew install it on OSX.

Below is an example how to take data in, parse JSON, modify it and then encode it back to string and return:

```javascript
#!/usr/bin/env node

process.stdin.resume();  
process.stdin.setEncoding('utf8');  
process.stdin.on('data', function(data) {
  var parsed_json = JSON.parse(data);
  // changing response
  parsed_json.response.status = 201;
  parsed_json.response.body = "body was replaced by JavaScript middleware\n";

  // stringifying JSON response
  var newJsonString = JSON.stringify(parsed_json);

  process.stdout.write(newJsonString);
});

```

You see, it's really easy to use it to create a synthetic service to simulate backend when you are working on the frontend side :)


### How middleware interacts with different modes

Each mode is affected by middleware in a different way. Since the JSON payload has request and response structures, some middleware
 will not change anything. Some information about middleware behaviour:
  * __Capture Mode__: middleware affects only outgoing requests.
  * __Virtualize Mode__: middleware affects only responses (cache contents remain untouched).
  * __Synthesize Mode__: middleware creates responses.
  * __Modify Mode__: middleware affects requests and responses.



## Debugging

You can supply "-v" flag to enable verbose logging.

## Contributing

Contributions are welcome!

To submit a pull request you should fork the Hoverfly repository, and make your change on a feature branch of your fork.
Then generate a pull request from your branch against master of the Hoverfly repository. Include in your pull request
details of your change (why and how, as well as the testing you have performed). To read more about forking model, check out
this link: [forking workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/forking-workflow).
Hoverfly is a new project, we will soon provide detailed roadmap.

## License

Apache License version 2.0 [See LICENSE for details](https://github.com/SpectoLabs/hoverfly/blob/master/LICENSE).

(c) [SpectoLabs](https://specto.io) 2015.

[CircleCI-Image]: https://circleci.com/gh/SpectoLabs/hoverfly.svg?style=shield
[CircleCI-Url]: https://circleci.com/gh/SpectoLabs/hoverfly
[ReportCard-Url]: http://goreportcard.com/report/spectolabs/hoverfly
[ReportCard-Image]: http://goreportcard.com/badge/spectolabs/hoverfly
