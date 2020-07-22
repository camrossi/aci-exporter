Promethues aci-exporter
------------

> This project is still in alpha and everything may change. If this exporter is useful or might be please share
> your experience and/or improvements.  

# Overview
The aci-exporter provide metrics from a Cisco ACI fabric by using the ACI Rest API against ACPI controller(s).

The exporter can return data both in the [Prometheus](https://prometheus.io/) and the [Openmetrics](https://openmetrics.io/) (v1) exposition format. 

# How to configure queries
 
The exporter provides two types of query configuration:

- Class queries - These are applicable where one query can result in multiple metric names sharing the same labels. 
- Compound queries - These are applicable where multiple queries result in single metric name with configured labels. 
This is typical when counting different entities.

> Example of queries can be found in the `example-config.yaml` file. 
> Make sure you understand the ACI api before changing or creating new ones.

## Class queries
Class queries can be done against the different ACI classes. For a single query multiple metrics can be collected. 
All metrics will share the same labels.  

Example of queries are:

- Node health of spine and leafs 
- Fabric health
- Tenant health
- Interface state

### Labels
Labels extraction is done by using regexp on one or more property from the json response using named expression.
In the below example we use the `topSystem.attributes.dn` property and parse it with the regexp 
`^topology/pod-(?P<podid>[1-9][0-9]*)/node-(?P<nodeid>[1-9][0-9]*)/sys` that will return label values for the label 
names `podid` and `nodid`. The property `topSystem.attributes.state` will return a label name `state` matching the
whole property value.

```
    labels:
      - property_name: topSystem.attributes.dn
        regex: "^topology/pod-(?P<podid>[1-9][0-9]*)/node-(?P<nodeid>[1-9][0-9]*)/sys"
      - property_name: topSystem.attributes.state
        regex: "^(?P<state>.*)"
```

## Compound queries 
The compound queries (better name?) is used when a single metrics is "compounded" by different queries. In the 
`example-config.yaml` file is an example where the number of spines, leafs and controllers are counted. They will
all be of the metric `nodes` but require 3 different queries. Since no labels can be extracted from the response 
the label name and label value is configured.

The result is:
```
# HELP nodes Returns the current count of nodes
# TYPE nodes gauge
aci_nodes{aci="ACI Fabric1",node="spine"} 3
aci_nodes{aci="ACI Fabric1",node="leaf"} 7
aci_nodes{aci="ACI Fabric1",node="controller"} 1
```

## Built-in queries  
The export has some standard metric "built-in". These are:
- Faults, labeled by severity and type of fault, like operational faults and configuration faults.

# Configuration

> For configuration please see the `example-config.yml` file.

All attributes in the configuration has default values, except for the fabric and the different query sections.
A fabric profile include the information specific to an ACI fabrics, like authentication and apic(s) url.

If there is multiple apic urls configured the exporter will use the first apic it can login to starting with the first
in the list.

# Openmetrics format
The exporter support [openmetrics](https://openmetrics.io/) format. This is done by adding the following accept header to the request:

    "Accept: application/openmetrics-text"

The configuration property `openmetrics` set to `true` will result in that all request will have an openmetrics 
response independent of the above header.

# Error handling
Any critical errors between the exporter and the apic controller will return 503. This is currently related to login 
failure and failure to get the fabric name.
 
There may be situations where the export will have failure against some api calls that collect data, due to timeout or
faulty configuration. They will just not be part of the output.

Any access failures to apic[s] are written to the log.

# Installation

## Build 
    go build -o build/aci-exporter  *.go

## Run
    .build/aci-exporter 
    
## Test
    curl -s 'http://localhost:8080/probe?target=my-fabric'
    
The target is a named fabric in the configuration file.
    
# Prometheus configuration

Please see file prometheus/prometheus.yml.

# Acknowledgements

Thanks to https://github.com/RavuAlHemio/prometheus_aci_exporter for the inspiration of the configuration of queries. 
Please check out that project especially if you like to contribute to a Python project.   

# License
This work is licensed under the GNU GENERAL PUBLIC LICENSE Version 3.
 
# Todo 
- Exclude configured queries for a specific fabric.
- Add fabric specific configuration queries
