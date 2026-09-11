This document will serve as a functional specification for building a test bed and demonstration for the upcoming CCNA 2.0 exam topics list.

For reference, this is being presented to Cisco Networking Academy instructors as part of their "Instructor Professional Development" week.  I will be co-presenting on this, but responsible for all demonstration of the actual "nuts and bolts" of this.

The CCNA 2.0 exam topics list is given here in PDF form: <https://learningcontent.cisco.com/documents/marketing/exam-topics/200-301_CCNA_v2.0_Exam_Topics_PDF.pdf>
THe previous version (of which the instructors are aware of currently) is here: <https://learningcontent.cisco.com/documents/marketing/exam-topics/200-301-CCNA-v1.1.pdf>

I am presenting on Domain 3.0 of the new blueprint.

My goal is to highlight the "hands on" portions of the exam topics (of which all items 3.1 - 3.4 exist).

Ideally, this will be done in a single CML topology using the nodes that are available in the CML-Free version of Cisco Modeling Labs (use this link as a reference: <https://developer.cisco.com/docs/modeling-labs/cml-free/> for what is supported in free version)

## DESIRED OUTCOME

- Single, CML-free compatible topology that highlights the components of CCNA 2.0 domain 3.0, including IPv4 and IPv6 OSPF, a single FHRP (use HSRP), and a floating static route, as well as routing table interpretation
- All IPv4 and IPv6 networks in OSPF should have full reachability within the topology
- Prefixes for exam topic 3.1 need not be reachable, as long as they appear in the routing table as entries for a user to interpret
- A floating static route on one router that will contain an error that will cause reachability to not happen when the OSPFv2 peering is shut down or a link is dropped (this should be a backup route and only using IPv4)
- Validate that all devices have external access over their MGMT VRF to a workstation that resides at 198.18.133.252.  Ping this device and report back on the status.

## DESIGN

- Use IOL, IOL-L2 devices as needed
- Use 5 active nodes (or under)
- External connectivity is not required, but would be a nice to have
  - External addressing should be provided by bridge0
  - External addressing should be in the 198.18.180.0 network with a /18 mask
  - Static routes should point to 198.18.128.1 for default gateway
  - Use a MGMT VRF attached to the first usable interface to keep the routing tables clean in global table
  - Use a breakout unmanaged switch to connect the MGMT interfaces to the external connector
- Routers can use any topology deemed appropriate to accomplish the tasks required
  - Links should include IPv4 and IPv6 addressing and use OSPFv2 and OSPFv3 to connect them
  - Include loopbacks on every device as additional items to see in the table
  - Router IDs should be of form 0.0.0.x, where x is the router number (R1, R2, R3, etc)
- The harder goal to simulate will be providing several different, overlapping routes using several different routing protocols (RIP, EIGRP, ISIS, eBGP for example)
  - The focus of this is to "clog up" the routing table with various overlapping prefixes of different prefix-lengths, combined with protocols which have different default "administrative distances", so that the user will need to use their knowledge of protocols and ADs to determine the path through the network.  
  - I understand that there is a finite number of "different paths" that can exist within a 5-router network, but the user should see some overlapping prefixes and ADs that I can demonstrate
  - If needed, use web search to find examples online of that existing topic item (CCNA 1.1 exam topic 3.1) for what to replicate
- A single pair of routers may be connected as part of an HSRP group.  Provide the bare minimum configuration for this to function, including group number, priority value to set one as primary vs. secondary, and pre-empt, as well as the shared VIP address.  This is "associate" level and there is no need for esoteric configuration of the FHRP.  We're just looking at status using show commands

## RESTRICTIONS/CAVEATS/GUARDRAILS

- Make all internal IP addressing (both IPv4 and IPv6) for the OSPF networks easy to understand.  Use an addressing schema that can be interpreted and inferred easily (/24s for IPv4, even for point-to-point links, /64s for IPv6) and change octets (say 3rd octet) to describe the routers being peered (.12 for R1<->R2, for example)
- All loopbacks should be carved from a single subnet, with the last octet indicating the router number
- Use OSPF point-to-point on some links, OSPF broadcast on others.  On links where there may be a DR/BDR election, you can set things like OSPF priority to ensure elections finish a given way.  On others, we may leave it up to the decision tree for who is elected
- Do not change any OSPF dead or hello timers or use any authentication.  This is out of scope for the CCNA

## OUTPUTS

- Save all configurations in a config folder when completed.  This folder should be at the root of this directory
- Create a README.md explaining what is occurring within the network and the way that an instructor can validate this on their own after this presentation,  This README should include:
  - Block diagram of the topology
  - All used addressing for p2p links, loopbacks, and RIDs in a table format (or whatever will look aesthetically pleasing on a GitHub page)
  - Validation steps for all IPv4 and IPv6 OSPF, along with expected outputs from various show commands
  - Interpretation of the "routing table" with the overlapping prefix and AD values
  - Status interpretation of the FHRP under normal state, as well as steps to act as forcing functions (shutting down interfaces, changing priority, validating pre-empt), etc
  - Discussion of the floating static route and why it fails along with other suggested activities that can be done to "play" with the floating route
  - If it is easier to have a single "front" README with basic instructions and getting started, with links to pages that are in a docs/ folder that cover each of the above items, this is also acceptable
- Save the CML YAML file at the root of this directory.  Ensure that both within CML and as it is saved that it includes mention of CCNA 2.0 Domain 3.0
