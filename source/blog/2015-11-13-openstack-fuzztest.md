---
title: Automated API testing workflow
author: TristanCacqueray
date: 2015-11-13 14:00:00 UTC
tags: testing
comments: true
published: true
---

Services exposed to a network of any sort are in risk of security exploits.
The API is the primary target of such attacks, and it is often abused by input that developers
did not anticipate.

This article introduces security testing of Openstack services using
[fuzzing techniques](https://en.wikipedia.org/wiki/Fuzz_testing#Techniques)

This article will demonstrate how OpenStack services could be automatically tested for security
defects.
First, I will introduce a set of tools called RestFuzz to demonstrate how it works in practice.
Second, I will discuss the limitations of RestFuzz and how more advanced techniques could be used.

The end goal is to design an automatic test to prevent a wide range of security bugs to enter
OpenStack's source code.


# How a fuzzer works

A fuzzer's purpose is to discover issues like
[OSSA 2015-012](https://security.openstack.org/ossa/OSSA-2015-012.html) by injecting random
inputs into the target program.
This advisory shows how a single invalid input, accepted by the API, broke the entire L2 network
service.
While code review and functional tests are not good at spoting such mistake, a fuzz test is a
proven strategies to detect security issues.

A fuzzer requires at least two steps: first, inputs need to be generated and fed to the
service's input; and second, errors needs to be detected.


## OpenStack API description

To reach the actual service code, a fuzzer will need to go through the REST interface routing,
which is based on HTTP method and url.
One way to effectively hit the interesting parts of the service code is to replay known valid
HTTP queries.
For example, a fuzzer could re-use HTTP requests created by
[tempest](http://git.openstack.org/cgit/openstack/tempest) functional tests.
However, this method is limited by its trace quality.
Instead, RestFuzz works the other way around and generates valid queries based on API definitions.
That way, it covers actions that functional tests do not use, such as unusual combinaisons of
parameters.

To keep the process simple, RestFuzz will need a description of the API that defines the methods'
endpoint and inputs' types.
While in theory the API description could be generated dynamically, it's easier to write it down
by hand.

As a C function description, a REST API method description could be written in YAML format like
this:

```yaml
  - name: 'network_create'
    url: ['POST', 'v2.0/networks.json']
    inputs:
      network:
        name: {'required': 'True', 'type': 'string'}
        admin_state_up: {'type': 'bool'}
    outputs:
      network_id: {'type': 'resource', 'json_extract': 'lambda x: x["network"]["id"]'}
```

This example from the [network api](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=api/network.yaml)
defines a method called "network_create" using:
* The HTTP method and url to access the API.
* The description of the method's parameters' name and types.
* The description of the expected outputs along with a lambda gadget to ease value extraction
  from JSON output.

The [method object](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=restfuzz/method.py)
shows how to call network_create based on the above description.
Note that the call procedure takes a dictionary as JSON inputs.
The next chapter will cover how to generate such inputs dynamically.


## Input generation

An input constructor needs to be implemented for each type.
The goal is to generate data known to cause application errors.


The [input_generator object](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=restfuzz/input_generator.py#l277)
shows the api call parameters can be dynamically generated by using the method input description
shown above. Here are a couple of input generators:

```python
    def gen_string():
        chunk = open("/dev/urandom").read(random.randint(0, 512))
        return unicode(chunk, errors='ignore')

    def gen_address_pair():
        return {
            "ip_address": generate_input("cidr"),
            "mac_address": generate_input("mac_address")
        }
```

Note that "resource" types are UUID data, which can't be randomly generated.
Thus, the fuzzer needs to keep track of method outputs to reuse valid UUID whenever possible.

The fuzzer can now call API methods with random inputs, but, it still needs to monitor
service API behavior.
This leads us to the second steps of error detection.


## OpenStack error detection

This chapter covers a few error detection mechanisms that can be used with OpenStack components.

### HTTP Error code

OpenStack services' API will return theses important error codes:

* 401 tells the keystone token needs to be refreshed
* 404 tells a UUID is no longer valid
* 500 tells unexpected error

### Tracebacks in logs

Finding tracebacks in logs is usually a good indicator that something went wrong.
Moreover, using file names and line numbers, a traceback identifier could be computed to
detect new unique errors.
The [health object](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=restfuzz/health.py)
features a collect_traceback function.

### API service process usage

Finaly cgroups can be used to monitor API services memory and CPU usage.
High CPU loads or constant growth of reserved memory are also good indicators that something
went wrong.
However this requires some mathematical calculations which are yet to be implemented.


## Fuzzing Workflow

All together, a fuzzing process boils down to:

* choose a random method
* call method using random inputs
* probe for defect
* repeat

The [ApiRandomCaller object](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=restfuzz/fuzzer.py)
uses random.shuffle to search for random methods and returns an Event object to the main process
[starting here](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=restfuzz/cmd.py#l60).


## RestFuzz features

Console output:

```bash
  $ restfuzz --api ./api/network.yaml --health ./tools/health_neutron.py
  [2015-10-30 06:19:09.685] port_create: 201| curl -X POST http://127.0.0.1:9696/v2.0/ports.json -d '{"port": {"network_id": "652b1dfa-9bcb-442c-9088-ad1a821020c8", "name": "jav&#x0D;ascript:alert('XSS');"}}' -> '{"port": {"id": 2fc01817-9ec4-43f2-a730-d76b70aa4ea5"}}'
  [2015-10-30 06:19:09.844] security_group_rule_create: 400| curl -X POST http://127.0.0.1:9696/v2.0/security-group-rules.json -d '{"security_group_rule": {"direction": "ingress", "port_range_max": 9741, "security_group_id": "da06e93b-87a3-4d90-877b-047ab694addb"}}' -> '{"NeutronError": {"message": "Must also specifiy protocol if port range is given.", "type": "SecurityGroupProtocolRequiredWithPorts", "detail": ""}}'
  [2015-10-30 06:19:10.123] security_group_create: 500| curl -X POST http://127.0.0.1:9696/v2.0/security-groups.json -d '{"security_group": {"name": ["Dnwpdv7neAdvhMUcqVyQzlUXpyWyLz6cW2NPPyA6E8Z9FbvO9mVn1hs30rlabVjtVHy6yCQqpEp0xcF1AsWZYAPThstCZYebxKcJaiS7o7fS0GsvG3i8kQwNzOl5F1SiXBxcmywqI9Y6t0ZuarZI787gzDUPpPY0wKZL69Neb87mZxObhzx4sgWHIRGfsrtYTawd6dYXYSeajEDowcr1orVwJ6vY"]}}' -> '{"NeutronError": {"message": "Request Failed: internal server error while processing your request.", "type": "HTTPInternalServerError", "detail": ""}}', /var/log/neutron/q-svc.log
  File "/opt/stack/neutron/neutron/api/v2/resource.py", line 83, in resource
    result = method(request=request, **args)
  File "/opt/stack/neutron/neutron/api/v2/base.py", line 391, in create
    allow_bulk=self._allow_bulk)
  File "/opt/stack/neutron/neutron/api/v2/base.py", line 652, in prepare_request_body
    attr_vals['validate'][rule])
  File "/opt/stack/neutron/neutron/extensions/securitygroup.py", line 195, in _validate_name_not_default
  ...
```

Network topology after one hour of fuzz test:

  ![Random Network Topology](/images/blog/restfuzz_network_topology.jpg)

Interesting features:

* [API description](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=tree;f=api) works for block storage (Cinder), image (Glance),
  network (Neutron) and dns (Designate).
* [Input generator](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=restfuzz/input_generator.py) randomly inter-changes types using
  a "chaos monkey" behavior to abuse the API even more.
* [Health plugin](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=restfuzz/health.py) implements custom probe per services.
* Simple workflow where methods are called in random order. An early version tried to build a
  graph to walk through the API more efficiently, but it turns out that calling method randomly
  is efficient enough and it keeps the code much more simple.
  See [this image](/images/blog/restfuzz_graph.jpg) for a dependency graph of the network API.
* More importantly, it already found a bunch of security defect like
  [bug #1486565](https://bugs.launchpad.net/neutron/+bug/1486565) or
  [bug #1471957](https://bugs.launchpad.net/neutron/+bug/1471957).
  The full list is available in the [README](http://softwarefactory-project.io/r/gitweb?p=restfuzz.git;a=blob;f=README).

However, RestFuzz is a proof of concept and it's not ready to be used as an efficient
gate mechanism.
The next part of this article discusses the limitations reached and what needs to be done
to get something awesome!


# Limitations and improvments

## It's built from the ground up

RestFuzz currently only requires python-requests and PyYAML.
A good chunk of the code could use frameworks such as the
[syntribos](http://git.openstack.org/cgit/openstack/syntribos) and/or
[gabbi](https://github.com/cdent/gabbi)


## API description

Managing API descriptions is a great limitation, as the fuzzer will only be as good as the API
description.
OpenStack really needs to provide us the API description in a unified and consistent format:

* [API Documentation](http://developer.openstack.org/api-ref.html) does not describe the input
  object precisely enough, making it impossible to use automatically.
* [Service documentation](http://git.openstack.org/cgit/openstack/nova/tree/doc/api_samples) is
  sporadic and REST API routing is done differently in most OpenStack projects.
* [Command line tools](http://developer.openstack.org/) may provide a good base, but they usualy
  omit unusual parameters and service extensions.

Having an unified API description will give a fuzzer bigger code coverage.


## Complex types generation

A formal description of API inputs is a real challenge.
Some services like Heat or Sahara can take flat files as input that would requires dedicated
input generators.
Otherwise, services like Designate need coherent inputs to pass early checks.
E.G., the domain name used at zone creation needs to be reused for record creation.

In theory, this limitation could be mitigated using better instruments like guided fuzzing.


## Guided fuzzing

A modern fuzzer like [American Fuzzy Lop](http://lcamtuf.coredump.cx/afl/) (afl) is able to
monitor which instructions are executed by the target program and then use
"rock-solid instrumentation-guided genetic algorithm" to produce inputs.
In short, afl can generate valid input out of thin air without any description.

However afl is difficult to put in place for network services because it requires services'
modification.
[python-afl](http://jwilk.net/software/python-afl) bridges the Python interpreter with the
fuzzer, and it may be interesting for OpenStack services testing.


## Anomalies detection probes

Better watchdog processes could be used in paralel to detect silent logic flaws.
Stacktraces and 500 errors most often reveal basic programing flaws.
If one of the following actions fail, it's a good sign something is defective:

* Boot an instance, assign an ip and initiate connection
* Test methods with expected output like with
  [gabbi](http://gabbi.readthedocs.org/en/latest/example.html)
* Run functional tests


## Fuzzer As A Service

A fuzzer could also be implemented as an OpenStack service that integrates nicely with the
following components:

* Compute resources with Nova
* Database store with Trove
* Message bus for parallel testing with Zaquar
* Data processing with Sahara/Sparks


# Conclusion

Fuzzing is a powerful strategy to pro-actively detect security issues.
Similar to [bandit](https://wiki.openstack.org/wiki/Security/Projects/Bandit) the end goal is
to have an automatic test to detect and prevent many kinds of code defect.

While RestFuzz is really just a naïve proof of concept, it nonetheless indicated important
limitations such as the lack of API description.
I would appreciate feedback and further would like to create an upstream community to implement
OpenStack fuzz testing.

This effort could start as an offline process with automatic reporting of traceback and 500
errors.
But ultimately this really needs to be used as a gating system to prevent erroneous commits to
be released.

I'd like to thank fellow Red Hat engineers Gonéri Le Bouder, Michael McCune, and George
Peristerakis for their support and early reviews.
