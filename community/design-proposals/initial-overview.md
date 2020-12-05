## The High-Level Picture
The more we explored this proxy idea, the more ideas I had about remote-data collection, and our eventual goal to provide Print Tracker as a simple self-hosted solution (like our competitors).

### Remote Data Collection
The biggest hurtle that we deal with is fine-tuning our data collection process which historically resulted in lots of iterative, remote-updates (very time-consuming). These inconveniences brought about the DGI and TEC architectures which aimed to define the data collection process as loadable configurations that could be tweaked and changed remotely. Even with these architectures, we're generally releasing upgrades 1-2 times per week.

Having a proxy mesh would replace a data collection agent on the customer network with a very basic proxy agent that would need minimal upgrades. The data collection agent would be moved to server-side, when a device scan needs to run, the agent would create a new tunnel object (shown below) and wait for the tunnel to be started by the locally-installed proxy agent before using the tunnel to scan the network.

### Virtual Network
The virtual network idea entails tunneling traffic for every IP address in the network. Once the tunnel is established, you could "mount" a data collection agent into the virtual network and perform remote device scans on an entire network at a time.

### Performance
My biggest concern with remote data collection is the added latency between request and response that would follow every request we made through the tunnel. In order to reduce the total runtime for a process such as collecting data from a network's worth of devices, we could look at allowing several requests to run concurrently. For example, instead of making a request for one OID value at a time, we could potentially allow up to 10 concurrent requests from a remote data collection agent at a time.

### Installation
The following process would connect and register and agent, as well as registering the network the agent is in charge of tunneling:
1. User enters a bootstrapping token into the agent that was obtained from the web portal.
2. The agent uses the bootstrapping token to initiate a CSR.
3. The agent registers the new certificate and opens the gRPC stream or polling collection using the new certificate.
4. The agent registers the expiration of the certificate and uses the bootstrapping token to create a new CSR when the certificate is about to expire. This process allows us to revoke the ability for new registrations from a specific user at any time as well as revoke their certificate. It also allows certificate rotation to happen often.
5. The agent uses something like https://github.com/hashicorp/serf to look for other registered agents on the network
    * If it does not find another agent, it generates a unique ID for the network it belongs to. This unique ID should be random, if it were determined by network topology I think we may get a lot of duplicates. This network ID is attached to the agent's registration.
    * If it does find another agent, it registers the same network ID as the first agent. This network ID allows tunnel objects to be created for a specific network ID but not a specific agent ID, allowing agents on the same network to load balance each other.
6. The agent begins a control loop that involves:
    * Watching for all tunnel objects that belong to the agent's network ID.
    * It will attempt to acquire a lease for any tunnel objects whose agent lease is either missing or expired.
    * For each lease the agent acquires, it will open a tunnel with the agents servers. The server will use the configuration of the tunnel object to determine how to expose/route traffic through the tunnel.

### Agent Server
#### Proxying 3rd Party Traffic
The agent server is not only responsible for exposing agent-initiated tunnels but also proxying traffic from 3rd parties to the appropriate instance of the agent server. For example, if `agent1` connects to `agent-server1` and a 3rd party request that is destined to go through `agent1`'s tunnel, but due to how the external load-balancer runs the request ends up being answered by `agent-server2`, then `agent-server2` will need to proxy that request to `agent-server1`.

## Tunnel Object
The tunnel object is stored in a database and provided to agents by request/stream. The tunnel object defines the current state of the tunnel, as well as the specifications for the tunnel. The object provides the ability for multiple servers to interact with potentially multiple agents that are candidates for the tunnel. The object also describes how the tunnel should be accessed, whether it can be accessed using an external endpoint, or whether it can only be accessed by internal services
