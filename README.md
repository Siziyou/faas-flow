# FaaSChain - FaaS pipeline as a function
[![Build Status](https://travis-ci.org/s8sg/faaschain.svg?branch=master)](https://travis-ci.org/s8sg/faaschain)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GoDoc](https://godoc.org/github.com/s8sg/faaschain?status.svg)](https://godoc.org/github.com/s8sg/faaschain)
[![OpenTracing Badge](https://img.shields.io/badge/OpenTracing-enabled-blue.svg)](http://opentracing.io)
[![OpenFaaS](https://img.shields.io/badge/openfaas-serverless-blue.svg)](https://www.openfaas.com)

> **Pure FaaS**   
> **Completely Stateless**   
> **Build on current `go` template**   
> **Available as a template `faaschain`**   
> **Lightweight, leverage `openfaas` platform's capabilities**   
     
## What is it ?
FaaSChain allow you to define your faas functions pipeline and deploy it as a function
![alt overview](https://github.com/s8sg/faaschain/blob/master/doc/overview.jpg)
     
## How does it work ?
FaaSChain runs four mejor steps to define and run the pipeline
![alt internal](https://github.com/s8sg/faaschain/blob/master/doc/internal.jpg)

| Step |  description |
| ---- | ----- |
| Build Chain | Identify a request and build a chain. A incoming request could be a partially finished pipeline or a fresh raw request. For a partial request `faaschain` parse and understand the state of the pipeline from the incoming request |
| Get Definition |  FaasChain create simple **pipeline-definition** with one or multiple phases based on the chain defined at `Define()` function in `handler.go`. A **pipeline-definition** consist of multiple `phases`. Each `Phase` includes one or more `Function Call`, `Modifier` or `Callback`. Always a single `phase` is executed in a single invokation of the chain. A same chain always outputs to same pipeline definition, which allows `faaschain` to be completly `stateless`|
| Execute | Execute executes a `Phase` by calling the `Modifier`, `Functions` or `Callback` based on how user defines the pipeline. Only one `Phase` gets executed at a single execution of `faaschain function`. |
| Repeat Or Response | If pipeline is not yet completed, FaasChain forwards the remaining pipeline with `partial execution state` and the `partial result` to the same `chain function` via `gateway`. If the pipeline has only one phase or completed `faaschain` returns the output to the gateway otherwise it returns `empty`| 

The `execution-state` is the execution position which denotes the next execution `phase` position. 

**`Async` function call results a chain to have multiple phases**
![alt single phase](https://github.com/s8sg/faaschain/blob/master/doc/asynccall.jpg)

**For only `Sync Apply` function call a chain create only one phase and return the result to the caller**
![alt multi phase](https://github.com/s8sg/faaschain/blob/master/doc/synccall.jpg)


   
| Acronyms |  description |
| ---- | ----- |
| Pipeline Definition | Definition which is generated for a chain. For a given chain the definition is always the same |
| Phase | Segment of a pipeline definiton which consist of one or more call to `Function`, `Modifier` or `Callback`. A pipeline definition has one or more phases. |
| Function | A FaaS Function. A function can be applied to chain by calling `chain.Apply(funcName)` or `chain.ApplyAsync(funcName)`. For each `Async` call a new phase is created.  |
| Modifier | A inline function. A inline modifier function can be applied as ```chain.ApplyModifier(func(data []byte) ([]byte, error) { return data, nil } )```. |
| Callback | A URL that will be called with the final/partial result. `chain.Callback(url)` |
  
## Example
https://github.com/s8sg/faaschain/tree/master/example


## Getting Started

#### **Get the `faaschain` template with `faas-cli`**.  
```
faas-cli template pull https://github.com/s8sg/faaschain
```
   
#### **Create a new `func` with `faaschain` template**.  
```bash
faas-cli new test-chain --lang faaschain
```
   
#### **Edit the `test-chain.yml` as:**.  
```yaml
  test-chain:
    lang: faaschain
    handler: ./test-chain
    image: test-chain:latest
    environment:
      gateway: "gateway:8080"
      chain_name: "test-chain"
      read_timeout: 120
      write_timeout: 120
      write_debug: true
      combine_output: false
      enable_tracing: true
      trace_server: "jaegertracing:5775"
```
> `gateway` : We need to tell faaschain the address of openfaas gateway
> ```
>              # swarm
>              gateway: "gateway:8080"
>              # k8
>              gateway: "gateway.openfaas:8080"
> ```
> `chain_name` : The name of the chain function. Faaschain use this to forward partial request.   
> `read_timeout` : A value larger than `max` phase execution time.     
> `write_timeout` : A value larger than `max` phase execution time.     
> `write_debug`: It enables the debug msg in logs.   
> `combine_output` : It allows debug msg to be excluded from `output`.  
> `enable_tracing` : It ebales the opentracing for requests and their phases.  
> `trace_server` : The address of opentracing backend jaeger.   

#### Start The Trace Server (jaeger - opentracing-1.x)   
To start the trace server we run `jaegertracing/all-in-one` as a service.  
```bash
docker service rm jaegertracing
docker pull jaegertracing/all-in-one:latest
docker service create --constraint="node.role==manager" --detach=true \
        --network func_functions --name jaegertracing -p 5775:5775/udp -p 16686:16686 \
        jaegertracing/all-in-one:latest
```
     
#### **Edit the `test-chain/handler.go`:**.  
```go
    chain.Apply("myfunc1", map[string]string{"method": "post"}, nil).
        ApplyModifier(func(data []byte) ([]byte, error) {
                log.Printf("Making data serialized for myfunc2")
                return []byte(fmt.Sprintf("{ \"data\" : \"%s\" }", string(data))), nil
        }).ApplyAsync("myfunc2", map[string]string{"method": "post"}, nil).
        Callback("storage.io/bucket?id=3345612358265349126")
```
> This function will generate two phases as:     
> ```
> Phase 1 :    
>     Apply("myfunc1")    
>     ApplyModifier()    
> Phase 2:    
>     ApplyAsync("myfunc2")   
>     Callback()    
> ```
#### **Build and Deploy the `test-chain` :**.  
Build
```bash
faas-cli build -f test-chain.yml
```
Deploy
```bash
faas-cli deploy -f test-chain.yml
```

#### **Invoke :**
```
cat data | faas-cli invoke -f test-chain.yml test-chain > updated_data
```
       
         
          
## Use async function
FaaSChain supports async function. It uses the `openfaas` async feature to implement.
To implement a async call use `ApplyAsync` function
#### **Edit the `test-chain/handler.go`:**.  
```go
    chain.Apply("myfunc1", map[string]string{"method": "post"}, nil).
         .ApplyModifier(func(data []byte) ([]byte, error) {
                log.Printf("Making data serialized for myfunc2")
                return []byte(fmt.Sprintf("{ \"data\" : \"%s\" }", string(data))), nil
        }).ApplyAsync("myfunc2", map[string]string{"method": "post"}, nil).
        Callback("http://gateway:8080/function/send2slack", 
                 map[string]string{"method": "post"}, 
                 map[string]string{"authtoken": os.Getenv(token)})
```
#### **Invoke :**
```
cat data | faas-cli invoke --async -f test-chain.yml test-chain
Function submitted asynchronously.
```

## Request Tracking by ID
Request can be tracked from the log by `RequestId`. For each new Request a unique `RequestId` is generated. 
```bash
2018/08/13 07:51:59 [request `bdojh7oi7u6bl8te4r0g`] Created
2018/08/13 07:52:03 [Request `bdojh7oi7u6bl8te4r0g`] Received
```

## Request Tracing by Open-Tracing 
Request tracing can be enabled by providing by specifying    
```yaml
      enable_tracing: true
      trace_server: "jaegertracing:5775"
```
Below is an example of tracing for an async request with 3 phases    
![alt multi phase](https://github.com/s8sg/faaschain/blob/master/doc/tracing.png)
    
> *Due to some unresolved issue we can't extend the req span over multiple phases    
> we use the waterfall model as an workaround    
> For more details visit : https://github.com/opentracing/specification/issues/81    
    
    
## TODO:
- [ ] Export support for Debug    
>      Request Execution Status.    
>      Execution Time.   
>      Pipeline Overview.   
- [ ] Support Of Multi Path Pipeline
