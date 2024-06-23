# Smart Downlink

- Start Date: 2024-06-23
- RFC PR: [#10](https://github.com/meshtastic/rfcs/pull/10)
- Affected Components: (e.g., Firmware, Network Management, all clients, etc.)

## Summary

Provide an additional option for downlinking to a channel to allow the extension of communication via MQTT without saturating the network.

## Motivation

If you currently would enable Downlink on a busy channel, your air time will be used up very quickly, your node list will be filled with nodes from all over the world and the network suffers a lot.

## Ecosystem Impact

This change will be possible without affecting users that don't have that feature yet as the decision to downlink a packet can be made by the router device itself.

## Protocol Buffer Changes

ChannelSettings will need to change from a simple boolean "downlink_enabled" towards an enumaration containing "all", "smart" and "none". "All" would replicate the current functionality if downlink is enabled, "none" would do the same for downlink disabled. "Smart" would apply the logic proposed in this RFC to packets.

## Technical Details

The device shall try to decrypt the packet to ensure that the MQTT packet really is intended for the channel it's configured to be on.
In addition to just checking if downlink is enabled, the device shall check if it has ever seen the destination node of the packet (check the NodeDB) and if that node has been seen a configurable amount of time ago.
If both checks pass, the device shall enqueue the packet for routing as normal. If one of the checks fail, the packet will be ignored.
Broadcast packets shall be ignored by default, this might be a candidate for making it configurable as well, e. g. for medium sized user groups that want to see other nodes join but generally lower the amount of air time by filtering out unneeded downlinks.

### Compatibility Considerations

This change can be implemented on any single device without impacting any others on the mesh.

### Security Considerations

None.

### Performance Considerations

This will create additional load on a route as decryption and checks aren't free. This will however be significantly less impactfull than allowing full downlink traffic.

## Drawbacks

None?

## Rationale and Alternatives

- This would allow e. g. direct messages to be sent through MQTT without the drawbacks of full downlinking

## Unresolved Questions

- Is it this easy? I've had a look at the code and it seems to be possible without too many changes in the firmware. However, I'm quite new to the project and wouldn't assume that this hasn't been ruled out already.
