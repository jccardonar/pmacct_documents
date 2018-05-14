# Introduction
Pmacct adapts different data sources to a single output model of flow data.  In this document, we aim to provide details on how the application, based on its configuration, extracts the information from these sources to populate BGP related fields.

# Short description of pmacct flow model
The output of pmacct corresponds to aggregated flow data. Data is aggregated in time and over flow attributes. Time aggregation is out of the scope of this document. Flow attribute aggregation reduce the number of output fields by aggregating the bytes/packets of the flows with the same characteristics. The aggregation fields that are used to define a flow can be configured with the "aggregate" keyword.

The next are the particular fields that we aim to describe here:
* BGP fields: 'src_as', 'dst_as', 'peer_src_as' , 'peer_dst_as', peer_dst_ip
* NET fields: 'dst_net', 'src_mask', 'dst_mask', 'src_net',
Most of these fields are self-descriptive. The only that requires further explanation is peer_dst_ip which is the BGP next-hop. The peer_dst_ip is valuable when one wants to find node-to-node, interface-to-interface, or peer-to-peer traffic matrices.

# Sources of data
The aforementioned fields can be populated using any of the flow protocols (netflow, sflow), but they can also be populated using bgp, bmp, files, etc. The configuration parameters used to select the data to populate the fields is controlled using nfacctd_as and nfacctd_net and nfacctd_as (sfacctd_as and sfacctd_net.if using sflow). Next, we describe how the fields are extracted from each of these protocols. **We assume that both fields are configured with the same value**.

The same procedure is performed for the src and dst ip. 

## Netflow/IPFIX
Nfacct supports Netflow version 5 and version 9. IPFIX behaves similarly to Netflow version 9.

### Netflow v5
Netflow v5 does not carry BGP information. Pmacct offers the use_ip_next_hop keyword, which commands nfacctd to use the ip_next_hop as source for the peer_dst_ip field.

### Netflow v9/IPFIX
Netflow v9 includes optional fields that contain the BGP data. Note that the information is optional, and even if device supports Netflow v9 it does not guarantee that it will populate these fields. 
TBD: Write how each of the fields map to the different pmacct fields?

## Sflow
TBD: Need to talk about nuances here.

## BGP
Pmacct can use the BGP entries of a peer to find BGP information for a flow.
The process to find the peer that matches a flow registry is:
- If the flow source corresponds to a BGP peer (either BGP ID or interface), pmacct uses that peer's BGP table.
- The bgp_agent_map can be used to relate a flow source to a bgp peer.

After Pmacct selects a BGP peer, it retrieves the paths for the flow by doing a lookup of the flow dst ip. If a (single) path is found, it is used to populate the BGP and Net fields. 

In cases in which multiple paths are available for the flow (for instance, when the BGP peer advertises multiple paths using the ADD-PATH capability), the result can be ambiguous. In this case, pmact TBD

## BMP
TBD

## Files
TBD ??
How is peer_dst_ip populated when a networks_file is done?

## Longest
Pmacct uses the src and dst nets from each of the  protocols to select the source of the data to populate the BGP and Net fields. If the prefix is equal, the preference becomes networks_file < sFlow/NetFlow < IGP <= BGP. Note that if some of the information from the source is empty, that will the value outputed by pmacct. 

Questions: Where is BMP here? Can the procedure differ from src and dst?

# Network scenarios
Let us discuss different network/monitoring scenarios base on the information of the previous sections.

## Multiple BGP paths received in the collector
BMP and BGP, using the ADD-PATH capability, can announce multiple paths for a specific prefix. However, the protocols still do not have the capability of signaling the paths installed in the FIB. Even in the case in which a single path is used for forwarding, announcing multiple paths to pmacct will make the result ambigous. It is recommended to only announce single paths if relying on BGP to populate BGP fields.

## Prefix forwarded using multiple BGP paths
In general, Networks with a heavy use of load balancing across BGP paths should rely on flow protocols populating BGP fields for correct accounting. BGP and BMP do not contain the mechanims to signal which BGP paths are used for forwarding. Even if future modifications of these protocols could signal the paths used for forwarding, allowing pmacct to add them all to the flow output, one would need to simulate the hashing algorithm of the router to infer the balancing among the paths.

## Using RRs to obtain BGP data
It is recommended to establish BGP sessions from every router exporting netflow. Some networks might want to reduce the number of BGP sessions by establishing them from their RR. To allow this, the bgp_agent_map could be used to link the edge routers to the RRs. This setup would work only if each edge router is mapped to a RR that would choose the same paths as the edge router.
