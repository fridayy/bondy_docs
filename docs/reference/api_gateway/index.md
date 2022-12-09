<script setup>
import { useData } from 'vitepress'
const { theme } = useData()
</script>


# Introduction
Bondy HTTP API Gateway is a reverse proxy that lets you manage, configure, and route incoming HTTP requests to your WAMP APIs or to external HTTP APIs. It allows Bondy to be easily integrated into an existing HTTP/REST API ecosystem.


## Overview
Bondy API Gateway is mainly used to expose a WAMP API via an HTTP (REST) API. However, it can also target external HTTP APIs i.e. acting as a traditional HTTP API Gateway.

Bondy API Gateway can host one or more HTTP (REST) APIs where each API can only be associated with a single [Realm](/concepts/realms).



::: warning Cluster deployments
APIs are replicated, synchronised and activated across the cluster automatically, so that each node serves all the defined APIs.
:::

## Defining an API

Each API is defined using an [API Gateway Specification](/reference/api_gateway/specification) document, a JSON data structure that _declaratively_ defines how Bondy should handle each HTTP Request e.g. convert into a WAMP operation or forward it to an external HTTP API. This includes capabilities for data transformation.

[API Gateway Specification](/reference/api_gateway/specification) declarations make use of a logic-less [domain-specific language expressions](/reference/api_gateway/expressions) (internally called _"mops"_) for data transformation and dynamic configuration.

## Loading and managing APIs

You can load, inspect and delete API Specifications using the [Admin HTTP API](/reference/http_api/api_gateway).



## Next Steps
<Features
    class="VPHomeFeatures"
    :features="theme.sidebar['/reference/api_gateway'][0].items.filter(function(item){return item.isFeature})"/>