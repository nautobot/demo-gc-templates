# nautobot-gc-templates

This is a demo repo that holds the Jinja2 templates Nautobot Golden Config renders into intended configurations.

The templates provided here are leveraging the the following GraphQL Query and transposer with `Shorten the SoT data returned` turned on.

```
query ($device_id: ID!) {
  device(id: $device_id) {
    config_context
    hostname: name
    position
    serial
    primary_ip4 {
      id
      primary_ip4_for {
        id
        name
      }
    }
    tenant {
      name
    }
    tags {
      name
    }
    role {
      name
    }
    platform {
      name
      manufacturer {
        name
      }
      napalm_driver
    }
    location {
      name
      vlans {
        id
        name
        vid
      }
      vlan_groups {
        id
      }
    }
    interfaces {
      description
      mac_address
      enabled
      mgmt_only
      name
      ip_addresses {
        address
        role {
          name
        }
        tags {
          id
        }
      }
      connected_circuit_termination {
        circuit {
          cid
          commit_rate
          provider {
            name
          }
        }
      }
      tagged_vlans {
        id
        vid
      }
      untagged_vlan {
        id
        vid
      }
      cable {
        termination_a_type
        status {
          name
        }
        color
      }
      tagged_vlans {
        location {
          name
        }
        id
      }
      tags {
        id
        name
      }
    }
  }
}
```

Transposer
```
"""Details."""
import ipaddress
# Can take the returned data from the graphql query and can modify the data before sending to the jinja2 template.
def transposer(data):
    """Some."""
    if len(data["location"]["vlans"]) > 0:
        data["vlans_merged_list"] = vlan_parser(
            [vlan["vid"] for vlan in data["location"]["vlans"]]
        )
    if data["platform"]["name"] == "Cisco IOS":
        expand_prefix(data["interfaces"])
    if data["platform"]["name"] == "Cisco NX-OS":
        format_mac_to_nxos(data["interfaces"])
    return data
    # return data["interfaces"]
def expand_prefix(interfaces):
    """Some."""
    for interface in interfaces:
        if len(interface["ip_addresses"]) > 0:
            for addr in interface["ip_addresses"]:
                addr["address"] = ipaddress.ip_interface(addr["address"]).with_netmask.replace("/", " ")
    return interfaces
def format_mac_to_nxos(interfaces):
    """Some."""
    for interface in interfaces:
        if interface.get("mac_address"):
            mac = ".".join(interface.get("mac_address").lower().replace(":", "")[i : i + 4] for i in range(0, 12, 4))
            interface["mac_address"] = mac
    return interfaces
# reference: https://github.com/ansible/ansible/blob/stable-2.9/lib/ansible/plugins/filter/network.py
def vlan_parser(vlan_list, first_line_len=48, other_line_len=44):  # pylint: disable=too-many-branches
    """Input: Unsorted list of vlan integer.

    Output: Sorted string list of integers according to cisco_ios-like vlan list rules
    1. Vlans are listed in ascending order
    2. Runs of 3 or more consecutive vlans are listed with a dash
    3. The first line of the list can be first_line_len characters long
    4. Subsequent list lines can be other_line_len characters
    """
    # Sort and remove duplicates
    sorted_list = sorted(set(vlan_list))
    if sorted_list[0] < 1 or sorted_list[-1] > 4094:
        raise ValueError("Valid VLAN range is 1-4094")
    parse_list = []
    idx = 0
    while idx < len(sorted_list):
        start = idx
        end = start
        while end < len(sorted_list) - 1:
            if sorted_list[end + 1] - sorted_list[end] == 1:
                end += 1
            else:
                break
        if start == end:
            # Single VLAN
            parse_list.append(str(sorted_list[idx]))
        elif start + 1 == end:
            # Run of 2 VLANs
            parse_list.append(str(sorted_list[start]))
            parse_list.append(str(sorted_list[end]))
        else:
            # Run of 3 or more VLANs
            parse_list.append(str(sorted_list[start]) + "-" + str(sorted_list[end]))
        idx = end + 1
    line_count = 0
    result = [""]
    for vlans in parse_list:
        # First line (" switchport trunk allowed vlan ")
        if line_count == 0:
            if len(result[line_count] + vlans) > first_line_len:
                result.append("")
                line_count += 1
                result[line_count] += vlans + ","
            else:
                result[line_count] += vlans + ","
        # Subsequent lines (" switchport trunk allowed vlan add ")
        else:
            if len(result[line_count] + vlans) > other_line_len:
                result.append("")
                line_count += 1
                result[line_count] += vlans + ","
            else:
                result[line_count] += vlans + ","
    # Remove trailing orphan commas
    for idx, value in enumerate(result):
        result[idx] = value.rstrip(",")
    # Sometimes text wraps to next line, but there are no remaining VLANs
    if "" in result:
        result.remove("")
    return result
```

The templates require `interfaces.mgmt_only` and `interfaces.tags.name`. If the query stored in Golden Config is missing any of them, rendering fails with an `UndefinedError` (Golden Config renders with `StrictUndefined`) instead of producing a config without management or OSPF handling.

## Interfaces the templates skip

`common/config_interfaces.j2` builds the list of interfaces the `interfaces.j2` templates loop over. It leaves out:

- Interfaces with `mgmt_only` set. Their real configuration (environment-specific addresses, DHCP, management VRFs) is not modelled in Nautobot, so rendering them would cut the device off its management network.
- `Loopback99`, the containerlab management loopback.

## Primary and secondary addresses

On EOS, IOS and NX-OS, an interface's first address is rendered as primary and the rest get `secondary`, in the order the Device Info query returns them.

## Management interface for services

`common/mgmt_state.j2` finds the device's management interface (the first `mgmt_only` interface). When there is one, syslog, SNMP, NTP and DNS are sourced from it, in the platform's management VRF:

| Platform | VRF | Rendered |
|---|---|---|
| EOS | default | `logging source-interface`, `snmp-server local-interface`, `ntp local-interface`, `ip domain lookup source-interface` |
| IOS | default | `logging source-interface`, `snmp-server trap-source`, `ntp source` |
| NX-OS | `management` | `use-vrf management` on syslog, SNMP, NTP and name servers, plus `logging source-interface`, `snmp-server source-interface traps`, `ntp source-interface` |
| Junos | `mgmt_junos` | `routing-instance mgmt_junos` on NTP servers and the trap group, plus `set snmp interface`. Syslog hosts and name servers get no `routing-instance`. Requires `set system management-instance` on the device |

Devices without a `mgmt_only` interface render none of these lines.

## Optional config context keys

Every key is optional. When a key is missing, the templates render the same lines they rendered before the key existed.

| Key | Shape | Renders |
|---|---|---|
| `dns` | `{domain: str, name_servers: [str]}` | Domain name and name servers on all platforms. |
| `syslog` | `{servers: [{ip: str, port: int}], port: int, level: str}` | Syslog hosts on all platforms. A server's `port` overrides the shared `port`. NX-OS turns `level` into its severity number. |
| `ospf` | `{process_id: int, area: str, reference_bandwidth: int}` | OSPF on every interface tagged `ospf`. `process_id` defaults to `1`, `area` to `0.0.0.0`. `reference_bandwidth` is in Mbps. |

### OSPF

- An interface runs OSPF only when it carries the `ospf` tag, has an address, and is one the templates configure (not `mgmt_only`, not `Loopback99`). Physical interfaces use the point-to-point network type. Loopbacks are passive on EOS, IOS and Junos; NX-OS has no passive option for loopbacks and does not send hellos from them.
- The router-id is the address of the tagged `Loopback0` (`loopback0` on NX-OS, `lo0` on Junos).
- A device without any tagged interface renders no OSPF configuration (including NX-OS `feature ospf`), even when `ospf` is defined.
