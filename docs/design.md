# 🎨 Key concepts and design

## <a name="foundations"></a> 🗿 Foundations

OPAL is built on the shoulders of open-source giants, including:
- [Open Policy Agent](https://www.openpolicyagent.org/)- the default policy agent managed by OPAL.
- [FastAPI](https://github.com/tiangolo/fastapi) - the ASGI API framework used by OPAL-servers and OPAL-clients.
- [FastAPI Websocket PubSub](https://github.com/permitio/fastapi_websocket_pubsub) - powering the live realtime update channels
- [Broadcaster](https://pypi.org/project/broadcaster/) allowing syncing server instances through a backend backbone (e.g. Redis, Kafka)

## <a name="key-concepts"></a>💡 Key Concepts
- ### OPAL is realtime (with Pub/Sub updates)
    - OPAL is all about easily managing your authorization layer in realtime.
    - This is achieved by a **Websocket Pub/Sub** channel between OPAL clients and servers.
    - Each OPAL-client (and through it each policy agent) subscribes to and receives updates instantly.
- ### OPAL is stateless
    - OPAL is designed for scale, mainly via scaling out both client and server instances; as such, neither are stateful.
    - State is retained in the end components (i.e: the OPA agent, as an edge cache) and data-sources (e.g. git, databases, API servers)

- ### OPAL is extensible
    - OPAL's Pythonic nature makes extending and embedding new components extremely easy.
    - Built with typed Python3, [pydantic](https://github.com/samuelcolvin/pydantic), and [FastAPI](https://github.com/tiangolo/fastapi) - OPAL is balanced just right for stability and fast development.
    - A key example is OPAL's FetchingEngine and FetchProviders.
    Want to use authorization data from a new source (a SaaS service, a new DB, your own proprietary solution)? Simply [implement a new fetch-provider](docs/HOWTO/write_your_own_fetch_provider.md).

## <a name="design"></a> ✏️ Design choices

- ### Networking
    - OPAL creates a highly efficient communications channel using [websocket Pub/Sub connections](https://github.com/permitio/fastapi_websocket_pubsub) to subscribe to both data and policy updates. This allows OPAL clients (and the services they support) to be deployed anywhere - in your VPC, at the edge, on-premises, etc.
    - By using **outgoing** websocket connections to establish the Pub/Sub channel most routing/firewall concerns are circumnavigated.
    - Using Websocket connections allows network connections to stay idle most of the time, saving CPU cycles for both clients and servers (especially when comparing to polling-based methods).

- ### Implementation with Python
    - OPAL is written completely in Python3 using asyncio, FastAPI and Pydantic.
    OPAL was initially created as a component of [Permit.io](https://www.permit.io), and we've chosen Python for development speed, ease of use and extensibility (e.g. fetcher providers).
    - Python3 with coroutines (Asyncio) and FastAPI has presented [significant improvements for Python server performance](https://www.techempower.com/benchmarks/#section=test&runid=7464e520-0dc2-473d-bd34-dbdfd7e85911&hw=ph&test=composite&a=2&f=zik0zj-qmx0qn-zhwum7-zijx1b-z8kflr-zik0zj-zik0zj-zijunz-zik0zj-zik0zj-zik0zj-1kv). While still not on par with Go or Rust - the results match and in some cases even surpass Node.js.

- ### Performance
    - It's important to note that OPAL **doesn't replace** the direct channel to the policy-engine - so for example with OPA all authorization queries are processed directly by OPA's Go based engine.

    - Pub/Sub benchmarks - While we haven't run thorough benchmarks **yet**, we are using OPAL in production - seeing its Pub/Sub channel handle 100s of events per second per server instance with no issue.

- ### Decouple Data from Policy
    - Open Policy Agent sets the stage for [policy code](https://www.openpolicyagent.org/docs/latest/rest-api/#policy-api) and [policy data](https://www.openpolicyagent.org/docs/latest/rest-api/#data-api) decoupling by providing separate APIs to manage each.
    - OPAL takes this approach a step forward by enabling independent update channels for policy code and policy data, mutating the policy agent cache separately.
    - **Policy** (Policy as code): is code, and as such is naturally maintained best within version control (e.g. git). OPAL allows OPA agents to subscribe to the subset of policy that they need directly from source repositories (as part of CI/CD or independently).
    - **Data**: OPAL takes a more distributed approach to authorization data - recognizing that there are **many** potential data sources we'd like to include in the authorization conversation (e.g. billing data, compliance data, usage data, etc etc). OPAL-clients can be configured and extended to aggregate data from any data-source into whichever service needs it.

- ### Decouple data/policy management from policy agents
    - OPAL was built initially with OPA in mind, and OPA is mostly a first-class citizen in OPAL. That said OPAL can support various and multiple policy agents, even in parallel - allowing developers to choose the best policy agent for their needs.

- ### <a name="large-scale-fga"></a> FGA, large scale / global authorization (e.g. Google Zanzibar)
    - OPAL is built for fine grained authorization (FGA), allowing developers to aggregate all and any data they need and restructure it for the authorization layer.
    - OPAL achieves this by making sure each policy-agent is loaded with only the data it needs via topic subscriptions (i.e: data focus and separation).
        - Examples of data separation: the back-office service doesn't need to know about customer users, a tenant specific service doesn't need the user list of other tenants, ...
    - That said OPAL is still limited by OPA's [resource utilization capacity](https://www.openpolicyagent.org/docs/latest/policy-performance/#resource-utilization).
      - If the size of the dataset you need to load into OPA cache is huge (i.e: > 5GB), you may opt to pass this specific dataset by [overloading input](https://www.openpolicyagent.org/docs/latest/external-data/#option-2-overload-input) to your policy.
      - OPAL can still help you if you decide to [shard your dataset](https://en.wikipedia.org/wiki/Shard_(database_architecture)) across multiple OPA agents. Each agent's OPAL-client can subscribe only to the relevant shard.
    - For these larger scale cases, OPAL can potentially become a link between a solution like Google Zanzibar (or equivalent CDN) and local policy-agents, allowing both Google-like scales, low latency, and high performance.
    - If you're developing such a service, or considering such high-scale scenarios; you're welcome to contact us, and we'd be happy to share our plans for OPAL in that area.

- ### Using OPAL for other live update needs
    - While OPAL was created and primarily designed for open-policy and authorization needs; it can be generically applied for other live updates and data/code propagation needs
    - If you'd like to use OPAL or some of its underlying modules for other update cases - please contact us (See below), we'd love to help you do that.

- ### Administration capabilities and UI
    - We've already built policy editors, back-office, frontend-embeddable interfaces, and more as part of [Permit.io](https://permit.io).
    - We have plans to migrate more parts of [Permit.io](https://permit.io) to be open-source; please let us know what you'd like to see next.