---
layout: post
title: Route leaking of remotely originated routes from VRF 
tags: junos
---
When designing networks, everyone must have run into a requirement where one needs to leak routes between various tables 

1. Leaking interface routes between tables [inet.0 -> inet.3 ]
2. Leaking PE-CE routes between tables [VRFA -> VRFB]
3. Leaking routes based on protocols [BGP based inet.0 -> inet.3]

How about scenarios where one needs to leak a remotely originated route ? There are 2 possible scenario one may run into depending on design

1. Far end PE originates the route which is received on a VRF  (VRF-A) and needs to be leaked to inet.0 ? i.e. VPN route leaked to inet.0 so that we can attract traffic from inet.0 and route it via VRF-A 
2. Far end PE originates the route which is received on a VRF (VRF-A) and this needs to be leaked to another VRF (VRF-B) so that we can advertise it to another PE depending on community. You can treat this as aggregate routes which needs to be advertised 

Let us talk through the scenarios

## Scenario #1 : Leak routes from VRF foo.inet.0 -> inet.0 for routes which are remotely originated 

Let us consider the below topology. Here vrf-foo on PE-1 has routes which are advertised over MPLS-VPN (inet-vpn unicast NLRI) to PE-2. The routes would be imported into vrf-foo of PE-2 based on vrf-import policy or vrf-target configuration. 

```
                             xxxxxxxx
                             x      xx
                       xxxxxxx       xxxxxxx
                       x                   xx
                    xxxx                    xxxxx
                   xx                           xx
         ┌─────────x                             ┌─────────┬───────┐
┌────────┤         │                             │         │vrf-foo│
│vrf-foo │  PE-1   │         MPLS Cloud          │  PE-2   ├───────┘
└────────┤         │                            x│  inet.0 │
         └─────────xxxx                     xxxxx└────┬────┘
                      xxx                  xx         │
                        xxxxxx        xxxxxx          │
                             xxx   xxxx               │
                               xxxxx                  │
                                                      │
                                                 ┌────┴────┐
                                                 │         │
                                                 │  dest   │
                                                 │         │
                                                 └─────────┘
```

Now, if you would like to leak this remotely originated route from vrf foo.inet.0 to inet.0, one can solve this in 2 ways 

### Option 1: 
1. leak route from vrf foo.inet.0 to inet.0 on pe-1 itself using interface routes or protocol based on pe-ce link leveraging rib-groups 
2. advertise the same route on both inet unicast and inet-vpn unicast NLRI 
3. routes would be received to PE-2 on both inet.0 table and foo.inet.0 table 
4. configure resolution ribs on PE-2 to resolve the route over foo.inet.0 and inet.3 instead of just resolving over inet.3

#### Notes
- configuration needed on far end router and is not local 
- routes need to be advertised from both tables and should be ensured its blocked from advertising to wrong PEs 

### Option 2: Leverage pRPD APIs to leak routes
1. Add a special community on routes which need to be leaked on PE-1 when advertising 
2. Advertise a loopback IP on PE-1 which will be used for route resolution and advertise to PE-2 with the special community
3. Run an on-box pRPD application on PE-2 which monitors vpn routes with the special community and add routes into inet.0 table with the loopback address as the protocol Next hop
4. Configure resolution ribs on PE-2 to resolve routes over foo.inet.0 and inet.3 instead of just resolving over inet.3 

#### Notes:
- No need to advertise route over inet-unicast from far end PE which needs additional config. i.e. config changes can be local to the router
- Route add/del are fast since they are ephemeral in nature. Interation is direct with RPD and by passes mgd
- No need for additional config. Things are handled dynamically. Configs can exist part of day 0/1
Download app from [here](https://github.com/ARD92/JET-scripts/tree/master/route-leak-vpn-to-inet)

### Option 3: Use lt- interface or a physical loopback between foo.inet.0 and inet.0 
1. configure an lt- interface and place each end in foo.inet.0 and inet.0 table 
2. Run BGP/other routing protocol on the link to advertise the routes 

#### Notes
- simple and leverage existing feature sets
- extra configuration
- burn 2 extra ports if using physical 
- consume fabric bandwidth when leveraging lt- interface. Lt- interfaces limitations apply


### Configuring resolution ribs example
```
set routing-options resolution rib inet.0 resolution-ribs foo.inet.0
set routing-options resolution rib inet.0 resolution-ribs inet.3
```

## Scenario 2: Leak routes from foo.inet.0 -> bar.inet.0 
Let us consider the below sceanario , where routes are advertised from foo.inet.0 on PE-1 to foo.inet.0 on PE-2. If we need to re-advertise this route to PE-3 with a different route target, there are two ways to solve 

```
                             xxxxxxxx
                             x      xx
                       xxxxxxx       xxxxxxx
                       x                   xx
                    xxxx                    xxxxx
                   xx                           xx
         ┌─────────x                             ┌─────────┬────────┐
┌────────┤         │                             │         │vrf-foo │
│vrf-foo │  PE-1   │         MPLS Cloud          │  PE-2   ├────────┤
└────────┤         │                            x│  inet.0 │vrf-bar │
         └─────────xxxx                     xxxxx└─────────┴────────┘
                      xxx                  xx
                        xxxxxx        xxxxxx
                             xxx   xxxx
                               xxxxx
                            ┌─────────┐
                            │         │
                            │  PE-3   │
                            │         │
                            └┬───────┬┘
                             │vrf-bar│
                             └───────┘
```

### Option 1: 2 VRF solution with static route with next-table 
1. Create a second VRF "bar" with route target matching the RT on PE-3 
2. Add a static route pointing to the routes advertised from PE-1 with next-table foo.inet.0 
3. Export the static route with RT so that it is advertised to PE-3

#### Notes:
- Routes when received on PE-2, will always land in bgp.l3vpn.0 and then be imported to foo.inet.0 table. This means the route imported into foo.inet.0 is a secondary route. This cannot be leaked again as only primary routes can be leaked. 
- Routes should be originated locally in order to be exported 
- The static route with next-table on bar.inet.0 will originate the route PE-2 and hence can be exported to PE-3 over inet-vpn unicast NLRI 

### Option 2: Single VRF solution with a generated route
- use a single VRF with vrf-import and vrf-export policies so that we can import only communities wrt vrf "foo". 
- use export policy with communities specific to "bar" so that it can be advertised to PE-3 
- generate a route on PE-2 within VRF on PE-2
    ```
    set routing-instances CUSTOMER routing-options generate route 44.44.44.0/24
    ``` 
- export this generated route from PE-2 to PE-3 
    ```
    set policy-options policy-statement CUSTOMER-EXPORT term 10 from protocol aggregate
    set policy-options policy-statement CUSTOMER-EXPORT term 10 then community add NEW-RT
    set policy-options policy-statement CUSTOMER-EXPORT term 10 then accept
    set policy-options community NEW-RT members target:300:300
    set routing-instances CUSTOMER vrf-export CUSTOMER-EXPORT
    ```

### Option 3: Leverage lt- interface or a physical loopback connecting both the VRFs 

1. configure an lt- interface and place each end in foo.inet.0 and inet.0 table 
2. Run BGP/other routing protocol on the link to advertise the routes 

#### Notes
- simple and leverage existing feature sets
- extra configuration
- burn 2 extra ports if using physical 
- consume fabric bandwidth when leveraging lt- interface. Lt- interfaces limitations apply
