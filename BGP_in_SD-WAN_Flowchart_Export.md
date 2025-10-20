# BGP Operations in Fortinet SD-WAN: Complete Flowchart Guide

## 1. BGP Route Advertisement Flow (Spoke to Hub)

This shows how a spoke advertises its local networks to the hubs, including the application of outbound route maps that can set communities, MED values, and other BGP attributes.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'14px'}, 'flowchart':{'useMaxWidth':false, 'htmlLabels':true}}}%%
flowchart TD
    Start(["Spoke has local network<br/>10.50.0.0/24"]) --> Network["BGP Network Statement<br/>or Redistribution"]

    Network --> OutMap{"Outbound Route Map<br/>configured?"}

    OutMap -->|Yes| OutRules["Apply Route Map Rules:<br/>- Set Communities<br/>- Set MED<br/>- Modify AS-Path<br/>- Set BGP Tags"]
    OutMap -->|No| SendHub

    OutRules --> Permit{"Rule Action?"}
    Permit -->|Permit| SendHub["Send to Hub1 & Hub2<br/>via BGP Update"]
    Permit -->|Deny| Drop["Route NOT Advertised"]

    SendHub --> Hub1["Hub1 Receives:<br/>10.50.0.0/24<br/>Community: 65001:10<br/>MED: 100<br/>AS-Path: 65050"]
    SendHub --> Hub2["Hub2 Receives:<br/>10.50.0.0/24<br/>Community: 65001:10<br/>MED: 100<br/>AS-Path: 65050"]

    Hub1 --> HubProcess["Hub Processing..."]
    Hub2 --> HubProcess

    style Start fill:#e1f5e1
    style Drop fill:#ffe1e1
    style SendHub fill:#e1e5ff
    style HubProcess fill:#fff5e1
```

### Key Points:

- **Network Statement:** Defines which local networks to advertise via BGP
- **Outbound Route Map:** Applied before sending to neighbors, modifies BGP attributes
- **Communities:** Tags for grouping routes (e.g., by site type or region)
- **MED:** Suggests to neighbors which path they should prefer
- **Redundancy:** Spokes advertise to both hubs for resilience

## 2. BGP Route Reception & Processing (At Hub)

When hubs receive routes from spokes, they apply inbound route maps that can filter routes and modify attributes before adding them to the BGP table.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'14px'}, 'flowchart':{'useMaxWidth':false, 'htmlLabels':true}}}%%
flowchart TD
    Receive["Hub Receives Route<br/>from Spoke"] --> InMap{"Inbound Route Map<br/>on neighbor?"}

    InMap -->|Yes| MatchRules["Match Conditions:<br/>- Prefix List?<br/>- Community List?<br/>- AS-Path Filter?<br/>- Origin?"]
    InMap -->|No| ToTable

    MatchRules --> Matched{"Matched?"}

    Matched -->|Yes| SetAttribs["Set Attributes:<br/>- Local-Preference<br/>- MED<br/>- Weight<br/>- Add/Remove Communities<br/>- Set BGP Tags"]
    Matched -->|No| NextRule{"More Rules?"}

    NextRule -->|Yes| MatchRules
    NextRule -->|No| DefaultAction{"Implicit Deny?"}

    DefaultAction -->|Yes| Reject["Route Rejected"]
    DefaultAction -->|No| ToTable

    SetAttribs --> RuleAction{"Rule Action?"}
    RuleAction -->|Permit| ToTable["Route Enters<br/>BGP Table"]
    RuleAction -->|Deny| Reject

    ToTable --> BestPath["BGP Best Path<br/>Selection Process"]

    style Receive fill:#e1f5e1
    style Reject fill:#ffe1e1
    style ToTable fill:#e1e5ff
    style BestPath fill:#fff5e1
```

### Key Points:

- **Inbound Route Map:** Applied when routes arrive from BGP neighbors
- **Match Conditions:** Prefix lists, community lists, AS-path filters determine which routes to process
- **Sequential Processing:** Route map rules evaluated in order until a match is found
- **Set Attributes:** Local-Preference, MED, Weight, Communities, and BGP Tags can be modified
- **Permit/Deny:** After matching and setting attributes, the rule action determines if the route is accepted
- **Implicit Deny:** Without a final permit rule, unmatched routes may be rejected (depends on configuration)

## 3. BGP Best Path Selection Process

This is the core algorithm BGP uses to choose the best path when multiple routes to the same destination exist. The algorithm is deterministic and follows a strict order.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'14px'}, 'flowchart':{'useMaxWidth':false, 'htmlLabels':true}}}%%
flowchart TD
    Start(["Multiple Paths for<br/>10.50.0.0/24 exist"]) --> Valid{"Path Valid?<br/>Next-hop reachable?"}

    Valid -->|No| Ignore["Ignore this path"]
    Valid -->|Yes| Weight{"Compare Weight"}

    Weight -->|Different| UseWeight["Use Highest Weight"]
    Weight -->|Same| LocalPref{"Compare<br/>Local Preference"}

    LocalPref -->|Different| UseLP["Use Highest<br/>Local-Pref"]
    LocalPref -->|Same| Local{"Locally<br/>Originated?"}

    Local -->|Yes| UseLocal["Prefer Local Route"]
    Local -->|Same/No| ASPath{"Compare<br/>AS-Path Length"}

    ASPath -->|Different| UseShorter["Use Shorter<br/>AS-Path"]
    ASPath -->|Same| Origin{"Compare Origin"}

    Origin -->|Different| UseIGP["IGP > EGP ><br/>Incomplete"]
    Origin -->|Same| MED{"Compare MED"}

    MED -->|Different| UseLowerMED["Use Lower MED"]
    MED -->|Same| PeerType{"eBGP vs iBGP?"}

    PeerType -->|Different| UseEBGP["Prefer eBGP"]
    PeerType -->|Same| IGPMetric{"IGP Metric to<br/>Next-Hop"}

    IGPMetric -->|Different| UseLowerIGP["Use Lower IGP Cost"]
    IGPMetric -->|Same| Age{"Oldest Route?"}

    Age -->|Different| UseOldest["Use Oldest Path<br/>for stability"]
    Age -->|Same| RouterID{"Lowest<br/>Router ID?"}

    RouterID --> UseLowerRID["Use Lowest<br/>Router ID"]

    UseWeight --> Winner
    UseLP --> Winner
    UseLocal --> Winner
    UseShorter --> Winner
    UseIGP --> Winner
    UseLowerMED --> Winner
    UseEBGP --> Winner
    UseLowerIGP --> Winner
    UseOldest --> Winner
    UseLowerRID --> Winner

    Winner(["Selected as<br/>BEST PATH"]) --> Install["Install in RIB<br/>Routing Table"]

    Install --> Advertise{"Advertise to<br/>other peers?"}

    style Start fill:#e1f5e1
    style Winner fill:#90EE90
    style Install fill:#e1e5ff

    classDef decision fill:#fff5cc
    class Weight,LocalPref,Local,ASPath,Origin,MED,PeerType,IGPMetric,Age,RouterID decision
```

### Key Points (In Order of Priority):

- **1. Weight:** Cisco/Fortinet local attribute (higher wins) - not standard BGP
- **2. Local-Preference:** Most important in SD-WAN! (higher wins, default 100)
- **3. Locally Originated:** Prefer routes you created yourself
- **4. AS-Path:** Shorter path wins (fewer AS hops)
- **5. Origin:** IGP (network cmd) > EGP > Incomplete (redistribute)
- **6. MED:** Lower wins (default 0) - only compared from same AS
- **7. eBGP > iBGP:** External peers preferred over internal
- **8. IGP Metric:** Lower cost to BGP next-hop wins
- **9. Age:** Older route preferred for stability
- **10. Router ID:** Tiebreaker - lowest router ID wins

## 4. Spoke Path Selection (Hub1 vs Hub2)

A practical example showing how a spoke chooses between two hub paths, and how that selection integrates with SD-WAN forwarding decisions.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'14px'}, 'flowchart':{'useMaxWidth':false, 'htmlLabels':true}}}%%
flowchart TD
    Start(["Spoke needs to reach<br/>Corporate Network<br/>172.16.0.0/16"]) --> Learn["Learns route from<br/>both Hub1 and Hub2"]

    Learn --> Hub1Path["Path via Hub1:<br/>Local-Pref: 200<br/>AS-Path: 65000<br/>MED: 100<br/>Next-hop: Hub1"]

    Learn --> Hub2Path["Path via Hub2:<br/>Local-Pref: 100<br/>AS-Path: 65000<br/>MED: 100<br/>Next-hop: Hub2"]

    Hub1Path --> Compare{"BGP Best Path<br/>Selection"}
    Hub2Path --> Compare

    Compare --> Step1{"Step 1: Weight?"}
    Step1 -->|Same Default 0| Step2{"Step 2: Local-Pref?"}

    Step2 -->|"Hub1: 200<br/>Hub2: 100"| Winner["Hub1 Path WINS!<br/>Due to Higher<br/>Local-Preference"]

    Winner --> SDWANRoute["Install in Routing Table"]

    SDWANRoute --> SDWANRule["SD-WAN Rules<br/>Evaluate Route"]

    SDWANRule --> Tunnel{"Which Tunnel<br/>to use?"}

    Tunnel --> Strategy{"SD-WAN Strategy?"}

    Strategy -->|SLA Based| CheckSLA["Check Tunnel SLA:<br/>- Latency<br/>- Jitter<br/>- Packet Loss"]
    Strategy -->|Volume Based| CheckLoad["Check Tunnel Load"]
    Strategy -->|Manual| UseStatic["Use Configured<br/>Priority"]

    CheckSLA --> SelectTunnel["Select Best Tunnel<br/>to Hub1"]
    CheckLoad --> SelectTunnel
    UseStatic --> SelectTunnel

    SelectTunnel --> Forward["Forward Traffic<br/>to Hub1"]

    style Start fill:#e1f5e1
    style Winner fill:#90EE90
    style Forward fill:#e1e5ff
```

### Key Points:

- **Local-Preference is King:** This is the primary tool for hub selection in SD-WAN
- **Set at Spoke:** Inbound route maps on spoke set different local-pref per hub
- **BGP to Routing Table:** Best path is installed in the RIB
- **SD-WAN Integration:** SD-WAN rules then use the routing table to make forwarding decisions
- **Tunnel Selection:** Even after BGP picks the hub, SD-WAN picks which tunnel to that hub based on SLA

## 5. Complete SD-WAN Scenario: Spoke to Datacenter Traffic

This comprehensive diagram shows the complete journey: from route advertisement, through hub processing, spoke path selection, and final traffic forwarding.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'14px'}, 'flowchart':{'useMaxWidth':false, 'htmlLabels':true}}}%%
flowchart TD
    Spoke(["Spoke Site:<br/>AS 65050<br/>10.50.0.0/24"]) --> |BGP Peering| Hub1["Hub1:<br/>AS 65000"]
    Spoke --> |BGP Peering| Hub2["Hub2:<br/>AS 65000"]

    Hub1 --> |BGP Peering| Core["Core Network:<br/>172.16.0.0/16"]
    Hub2 --> |BGP Peering| Core

    subgraph "Step 1: Route Advertisement"
        A1["Spoke advertises<br/>10.50.0.0/24"] --> A2["Outbound Route Map:<br/>Set Community 65001:50<br/>Set MED 100"]
        A2 --> A3["Sent to Hub1 & Hub2"]
    end

    subgraph "Step 2: Hub Processing"
        B1["Hub1 receives route"] --> B2["Inbound Route Map:<br/>Match Community 65001:50<br/>Set Local-Pref 150"]
        B2 --> B3["Install in BGP table"]

        B4["Hub2 receives route"] --> B5["Inbound Route Map:<br/>Match Community 65001:50<br/>Set Local-Pref 150"]
        B5 --> B6["Install in BGP table"]
    end

    subgraph "Step 3: Core Route Learning"
        C1["Hub1 advertises<br/>172.16.0.0/16 to Spoke"] --> C2["Outbound Route Map:<br/>Set MED 50<br/>Next-hop-self"]

        C3["Hub2 advertises<br/>172.16.0.0/16 to Spoke"] --> C4["Outbound Route Map:<br/>Set MED 150<br/>Next-hop-self"]
    end

    subgraph "Step 4: Spoke Path Selection"
        D1["Spoke receives routes<br/>from both Hubs"] --> D2["Inbound Route Map on Hub1:<br/>Set Local-Pref 200"]
        D1 --> D3["Inbound Route Map on Hub2:<br/>Set Local-Pref 100"]

        D2 --> D4{"BGP Decision:<br/>Local-Pref 200 vs 100"}
        D3 --> D4

        D4 --> D5["Hub1 Path Selected!<br/>Best Path to 172.16.0.0/16"]
    end

    subgraph "Step 5: SD-WAN Forwarding"
        E1["SD-WAN Rule evaluates<br/>destination 172.16.0.0/16"] --> E2["Matches BGP route<br/>via Hub1"]
        E2 --> E3["Check available tunnels<br/>to Hub1"]
        E3 --> E4{"SLA Metrics OK?"}
        E4 -->|Yes| E5["Forward via<br/>Primary Tunnel"]
        E4 -->|No| E6["Forward via<br/>Backup Tunnel"]
    end

    style Spoke fill:#e1f5e1
    style Hub1 fill:#e1e5ff
    style Hub2 fill:#e1e5ff
    style Core fill:#ffe1e1
    style D5 fill:#90EE90
    style E5 fill:#90EE90
```

### Complete Flow Summary:

- **Step 1:** Spoke tags and advertises its local networks to both hubs
- **Step 2:** Hubs receive, match communities, and install routes
- **Step 3:** Hubs advertise core/datacenter routes back to spoke with different MEDs
- **Step 4:** Spoke uses local-preference to select Hub1 as preferred path
- **Step 5:** SD-WAN forwarding uses the BGP route and selects best tunnel based on SLA

## 6. Route Map Decision Tree

Route maps are evaluated sequentially, rule by rule. This shows how a route progresses through multiple rules until it's either accepted with modifications or rejected.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'14px'}, 'flowchart':{'useMaxWidth':false, 'htmlLabels':true}}}%%
flowchart TD
    Start(["Route Map Evaluation"]) --> Rule1{"Rule 10"}

    Rule1 --> Match1{"Match Condition?"}
    Match1 -->|Prefix List: DATACENTER| Set1["Set Actions:<br/>Local-Pref 200<br/>Community 65000:100"]
    Match1 -->|No Match| Rule2

    Set1 --> Action1{"Rule Action?"}
    Action1 -->|Permit| Accept1["Route Accepted<br/>with Modifications"]
    Action1 -->|Deny| Reject["Route Rejected"]

    Rule2{"Rule 20"} --> Match2{"Match Condition?"}
    Match2 -->|Prefix List: BRANCH| Set2["Set Actions:<br/>Local-Pref 150<br/>Community 65000:200"]
    Match2 -->|No Match| Rule3

    Set2 --> Action2{"Rule Action?"}
    Action2 -->|Permit| Accept2["Route Accepted<br/>with Modifications"]
    Action2 -->|Deny| Reject

    Rule3{"Rule 30"} --> Match3{"Match Condition?"}
    Match3 -->|Any| Set3["Set Actions:<br/>Local-Pref 100"]

    Set3 --> Action3{"Rule Action?"}
    Action3 -->|Permit| Accept3["Route Accepted<br/>with Default Values"]
    Action3 -->|Deny| Reject

    Match3 -->|No More Rules| Implicit{"Implicit Deny?"}
    Implicit -->|Default Behavior| Reject

    Accept1 --> Applied["Route Map Applied<br/>Continue Processing"]
    Accept2 --> Applied
    Accept3 --> Applied

    style Start fill:#e1f5e1
    style Accept1 fill:#90EE90
    style Accept2 fill:#90EE90
    style Accept3 fill:#90EE90
    style Reject fill:#ffe1e1
```

### Key Points:

- **Sequential Processing:** Rules evaluated in numerical order (10, 20, 30, etc.)
- **First Match Wins:** Once a rule matches and permits/denies, processing stops
- **Match Then Set:** If match conditions are met, set actions are applied
- **Permit vs Deny:** After setting attributes, the rule action determines if route is accepted
- **Implicit Deny:** Routes that don't match any rule may be denied (depends on config)
- **Catch-All Rule:** Last rule typically matches "any" to provide default behavior

## 7. Hub Failover Scenario

What happens when your primary hub fails? This shows the BGP convergence process and how traffic automatically fails over to the backup hub, then reconverges when the primary recovers.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'14px'}, 'flowchart':{'useMaxWidth':false, 'htmlLabels':true}}}%%
flowchart TD
    Normal(["Normal Operation:<br/>All traffic via Hub1"]) --> Failure{"Hub1 Tunnel<br/>Fails"}

    Failure -->|Tunnel Down| BGPDead{"BGP Session<br/>Status"}

    BGPDead -->|"Hold Timer Expires<br/>180 seconds default"| Withdraw["Hub1 Path Withdrawn<br/>from BGP Table"]

    Withdraw --> Reconverge["BGP Reconvergence"]

    Reconverge --> NewBest["Hub2 Path now<br/>BEST PATH"]

    NewBest --> Update["Update Routing Table"]

    Update --> SDWANSwitch["SD-WAN switches<br/>to Hub2 tunnels"]

    SDWANSwitch --> Traffic["Traffic flows<br/>via Hub2"]

    Traffic --> Monitor{"Monitoring Hub1"}

    Monitor --> Recovery{"Hub1 Recovers?"}

    Recovery -->|Tunnel Up| BGPUp["BGP Session<br/>Re-establishes"]

    BGPUp --> ReadvertiseHub1["Hub1 Advertises<br/>Routes Again"]

    ReadvertiseHub1 --> CompareAgain["BGP Compares:<br/>Hub1 Local-Pref 200<br/>Hub2 Local-Pref 100"]

    CompareAgain --> BackToHub1["Hub1 Becomes<br/>Best Path Again"]

    BackToHub1 --> PreemptTime{"Preemption<br/>Delay?"}

    PreemptTime -->|Optional Timer| Wait["Wait for Stability"]
    PreemptTime -->|Immediate| Switchback

    Wait --> Switchback["Traffic Switches<br/>Back to Hub1"]

    Switchback --> Normal

    style Normal fill:#90EE90
    style Failure fill:#FFD700
    style Withdraw fill:#FFA500
    style SDWANSwitch fill:#e1e5ff
    style BackToHub1 fill:#90EE90
```

### Key Points:

- **Failure Detection:** BGP hold timer (default 180s) must expire before routes are withdrawn
- **Faster Detection:** Consider tuning BGP timers (e.g., 10/30) for quicker failover
- **Automatic Failover:** BGP automatically selects next-best path (Hub2)
- **Zero Touch Recovery:** When Hub1 returns, BGP automatically reconverges
- **Preemption:** Can add dampening timers to prevent flapping during unstable periods
- **SD-WAN Integration:** BGP provides the path, SD-WAN provides tunnel-level resilience

## 8. Component Relationships

This high-level diagram shows how all the BGP components interconnect and influence each other in an SD-WAN deployment.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'14px'}, 'flowchart':{'useMaxWidth':false, 'htmlLabels':true}}}%%
flowchart TD
    subgraph "Route Manipulation"
        RM["Route Maps"]
        PL["Prefix Lists"]
        CL["Community Lists"]
        ASF["AS-Path Filters"]

        RM -->|Uses| PL
        RM -->|Uses| CL
        RM -->|Uses| ASF
    end

    subgraph "BGP Attributes Set by Route Maps"
        LP["Local Preference"]
        MED["MED/Metric"]
        COMM["Communities"]
        TAG["BGP Tags"]
        WT["Weight"]

        RM -->|Sets| LP
        RM -->|Sets| MED
        RM -->|Sets| COMM
        RM -->|Sets| TAG
        RM -->|Sets| WT
    end

    subgraph "Path Selection Uses"
        LP --> BPS["BGP Best Path<br/>Selection"]
        MED --> BPS
        WT --> BPS
        ASP["AS-Path Length"] --> BPS
    end

    subgraph "Traffic Engineering"
        BPS --> RIB["Routing Table"]
        RIB --> SDWAN["SD-WAN Rules"]
        SDWAN --> FWD["Traffic Forwarding"]
    end

    subgraph "Administrative"
        COMM -->|Grouping| Policy["Policy Decisions"]
        TAG -->|Tracking| Policy
        Policy --> RM
    end

    style RM fill:#FFD700
    style BPS fill:#90EE90
    style SDWAN fill:#e1e5ff
```

### Understanding the Relationships:

- **Route Maps are Central:** They're the configuration tool that ties everything together
- **Match Tools:** Prefix lists, community lists, and AS-path filters are used by route maps to identify potential routes
- **Set Actions:** Route maps modify BGP attributes that influence path selection
- **Best Path Algorithm:** Uses the attributes set by route maps to choose optimal paths
- **Communities & Tags:** Used for administrative purposes and policy decisions
- **Final Output:** Best paths go into routing table, which SD-WAN uses for forwarding

## Usage Notes

1. **Route Advertisement**: Shows how routes leave a spoke with attributes
2. **Route Reception**: How hubs process incoming routes with route maps
3. **Best Path Selection**: The complete algorithm BGP uses
4. **Spoke Decision**: How spokes choose between multiple hubs
5. **Complete Flow**: End-to-end traffic flow with all components
6. **Route Map Logic**: How route map rules are evaluated sequentially
7. **Failover**: What happens when primary path fails
8. **Relationships**: How all components interconnect
