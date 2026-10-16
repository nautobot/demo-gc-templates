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

## Interfaces the templates configure

The `interfaces.j2` templates configure every interface the Device Info query returns, management ports included, so Nautobot must hold exactly the interfaces the device runs. In the containerlab lab, `Loopback99` exists only on IOS and NX-OS, where it carries the primary IP; on EOS and Junos the primary IP is on the management port and the device has no `Loopback99`.

Management ports render from their Nautobot address, so Nautobot must hold the address the device runs; otherwise remediation rewrites the device's management access. Their VRF is not read from Nautobot: it is fixed per platform by `mgmt_state.vrf` in the platform template (see the table under [Management interface for services](#management-interface-for-services)).

| Platform | Port | Rendered |
|---|---|---|
| EOS | `Management0` (`eos/_mgmt.j2`) | Description, `vrf <mgmt_state.vrf>` when the platform sets one (none on EOS), addresses |
| IOS | `GigabitEthernet1` (`ios/_physical.j2`) | Same stanza as the data ports, in the global table |
| NX-OS | `mgmt0` (`nxos/_mgmt.j2`) | Description, `vrf member management`, addresses |
| Junos | `fxp0` (`junos/_mgmt.j2`) | Description and addresses on `unit 0`. No VRF line: `set system management-instance` puts `fxp0` in `mgmt_junos` implicitly |

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
| `mgmt_gateway` | `str` | The management default route `0.0.0.0/0` via this next hop, in the main and startup templates: in the global table on EOS and IOS, under `vrf context management` on NX-OS, in `mgmt_junos` on Junos. |

### OSPF

- An interface runs OSPF only when it carries the `ospf` tag and has an address. Physical interfaces use the point-to-point network type. Loopbacks are passive on EOS, IOS and Junos; NX-OS has no passive option for loopbacks and does not send hellos from them.
- The router-id is the address of the tagged `Loopback0` (`loopback0` on NX-OS, `lo0` on Junos).
- A device without any tagged interface renders no OSPF configuration (including NX-OS `feature ospf`), even when `ospf` is defined.

## Startup config

`startup/<network_driver>.j2` renders the startup config of a containerlab device: the configuration it boots with and that the Containerlab Startup Reset job (solitaire-data) replaces the device configuration with. Golden Config does not render these templates itself, since it keeps one intended config per device; the job renders them with the device's Golden Config setting (its GraphQL query and this repository), then pushes the result with a configuration replace. The main templates render the change config on top of it.

The startup config holds:

- Hostname, AAA and local users.
- The management port (same partials as the main templates), its VRF, and `Loopback99` on IOS and NX-OS.
- The management default route via `mgmt_gateway`, set in the platform contexts in demo-git-datasource: `10.0.0.2` on IOS, NX-OS and Junos, the gateway of the QEMU user network vrnetlab gives the VM (the device is always `10.0.0.15/24`), and `172.18.0.1` on EOS, the containerlab management network gateway, since cEOS runs natively.
- What the device needs to stay manageable: EOS `management api`, IOS `archive` and `ip scp server enable`, NX-OS `feature nxapi`/`scp-server`, Junos `system services`.
- Every other physical port, bare: no description, addresses or OSPF. Ports keep their admin state from Nautobot, as in the main templates: EOS and IOS render `shutdown` and Junos `disable` on disabled ports, and NX-OS renders `no shutdown` on enabled ports; a disabled NX-OS port renders without `shutdown`, because the port is shut by default and the device does not show it. The lab enables only cabled NX-OS ports (Seed Lab Environment in solitaire-data).

It leaves out the change config (DNS, logging, NTP, SNMP, OSPF, `Loopback0`, port addresses), BGP and the other static routes, and the IOS self-signed certificate and license lines, which are unique to each device.

The templates require the `startup` config context key, which only the containerlab devices carry (`lab_startup.yml` in demo-git-datasource):

| Key | Shape | Renders |
|---|---|---|
| `startup` | `{secrets_group: str}` | Local users from the secrets group's username and password. |

Users render as Jinja placeholders such as `{{ "golden-config" | get_secret_by_secret_group_name("password") | safe }}`, which the Containerlab Startup Reset job in solitaire-data fills in with Golden Config's `get_secret_by_secret_group_name` filter before the config is pushed. NX-OS keeps `admin` (it cannot be removed) with the secrets group password; Junos keeps the user's password hash from the lab's startup config (it must match the secrets group password, since netutils cannot hash it without `crypt`/`legacycrypt`), and its `root-authentication` is unchanged.

The password is rendered unquoted after `secret 0` / `password 0`, so it must not contain spaces or `#`.
