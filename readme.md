# **What is the orchestrator**

It's a simple API written in python that runs steps configured in yaml files.

The orchestrator runs workflows, and each workflow is like a separate HTTP endpoint.

When an incoming request matches with the configurations (path + verbs), the workflow steps run sequentially, mirroring the results of the data collected during the execution of the steps. 

![](docs/result.jpg)

# **How it works**

## The Workflow configuration

The configuration is a yaml structure that must be placed in the `workflows/` directory

A workflow file looks like the following example:

[examples/weather.yml](examples/weather.yml)

```yaml
workflow:
  name: weather
  http:
    path: /weather
    verbs: [ 'get' ]

  steps:
    - name: Get Weather
      request:
        get: https://wttr.in/Lisbon?format=3

    - name:  Datetime + Weather
      result: 
        - ${{ str(datetime) }}
        - ${{ workflow.steps[1].result.content }}
```

The curly braces `${{ ... }}` allow you to run small python scripts. It is very useful when it comes to playing with data transformation and formatting the output result.

The content of the curly braces runs within a `context` where you can find variables and functions for manipulating data.

Example:

* **python `fstrings`:**
  
  >${{ f"`{http.url}`/my-url/"} }} 

* **incomming workflow data (http post):**

  >${{ fromjson( `http.data` ) }}

* **accessing environment variables**

  >${{ workflow.env.PATH }}

* **accessing workflow data**

  >${{ workflow.steps[0].name }}


## **List of builtin functions and variables**

### **variables**

* `workflow` :  the current workflow configuration parsed as object

* `env` :  environment variables

* `http` :  the incomming http request object (orchestrator) 

    >`url` : http request full url

    >`host` : http request host name

    >`scheme` : http request scheme

    >`path` : http request path

    >`headers` : http request headers

    >`data` : http request post raw data (must be deserialized if json)

* `time` :  the python time module
    
    >https://docs.python.org/3/library/time.html

* `datetime` :  the python datetime module
    
    >https://docs.python.org/3/library/datetime.html


### **functions**

* `serializable()` :      converts an object to dictionary to be serializable

* `fromjson()` :          deserializes a json string to object

* `tojson()`:             serializes an object to json 

* `fromxml()` :           serializes an object to xml 

* `toxml()`               deserializes a xml string to object

* `object()`            converts a dictionary to object


## **Workflow Documentation**

```yaml
workflow:
  name: <string>               # just a name
  http:
    path: <string>             # url path that triggers the workflow (eg.: /get/data )
    verbs:                     # http verb/method of the workflow
      - 'put' 
      - 'get' 
      - 'post' 
      - 'delete' 

  steps:

    # STEP TYPE CMD
    - name: <string>            # just a name
      async: <bool>             # if true, the cmd will be async  (default is false)
      hidden: <bool>            # if true, the step result will be omitted from the response  (default is false)
      cmd:                      # the step type, use cmd to create a terminal step type
        powershell: <string> 
        /bin/sh: <string>
        python: <string>
        (binary): <arguments>

    # STEP TYPE HTTP
    - name: <string>
      async: <bool>         # if true, the cmd will be async (default is false)
      hidden: <bool>        # omitt the step result from the response (default is false)
      request:              # the step type, use request to create an http step type
        post: <string>      # the http request method [get, put,post,delete] and the url
        data: <string>      # the request body, use the plugin ${{ tojson(obj)}} if you want to serialize an object to json
        headers:            # the http request headers (key-value pairs)
          Content-Type: <string>    

    # STEP TYPE SQL
    - name: <string>
      driver: <string>      #optional driver according go the database engine (default is MSSQL)
      hidden: <bool>        # omitt the step result from the response (default is false)
      cstring: <string>
      sql: |
        SELECT TOP 10 * 
          FROM [Localization].[dbo].[Localizations] 
          
    # STEP TYPE GENERIC
    - name: Output          # if the step type is not provided, it's just data
      hidden: true          # omitt the step result from the response (default is false)
      result:                
        - ${{ str(datetime.now()) }}
        - ${{ workflow.steps[0].result.content }}

    - name: Output2                  
      result: ${{ workflow.steps[3].result[0] }}
```


# Run it on docker

```powershell
docker build . -t orchestrator
```

```powershell
docker run -it -p 5000:5000 -v "$(PWD)/examples:/app/workflows"  jsoliveira/orchestrator
```
