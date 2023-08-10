# nautobot-gc-templates

This is demo repo to hold backup configurations for the purpose nautobot golden config plugin.

The templates provided here are leveraging the the following GraphQL Query and transposer with `Shorten the SoT data returned` turned on.

```
query ($device: String!) {
  devices(name: $device) {
    config_context
    name
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
      slug
    }
    device_role {
      name
    }
    platform {
      name
      slug
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
      name
      cpf_ntc_description
      ip_addresses {
        address
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
      }
      untagged_vlan {
        id
      }
      cable {
        termination_a_type
        status {
          name
        }
        color
      }
      tagged_vlans {
        site {
          name
        }
        id
      }
      tags {
        id
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
    if len(data["site"]["vlans"]) > 0:
        data["vlans_merged_list"] = vlan_parser(
            [vlan["vid"] for vlan in data["site"]["vlans"]]
        )
    if data["platform"]["slug"] == "cisco_ios":
        expand_prefix(data["interfaces"])
    if data["platform"]["slug"] == "cisco_nxos":
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
