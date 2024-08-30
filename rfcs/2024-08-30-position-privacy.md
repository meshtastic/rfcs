# MQTT and Position Privacy Features

- Start Date: 8/30/24
- RFC PR: 
- Affected Components: Firmware, clients, MQTT

## Summary

Add home zones, where the user can pick a position on the map, and a precision value, to send limited precision packets while in the zone.
Additionally, add a DoNotMQTT bit to the Data protobuff, to indicate that other nodes should not forward packets to an MQTT server, default true.

## Motivation

It's too easy for users to doxx themselves. While recent changes have improved things, the addition of home zones would allow a user even more control over what location data is sent.
An ongoing issue is that user data, including location data, is getting picked up by other nodes and forwarded to MQTT servers without their permission. While a DoNotMqtt bit is *technically* easy to circumvent, 
it does raise the bar for doing so. In addition, responsible public MQTT server owners could check for this bit and drop messages that contain it on public key messages.

## Ecosystem Impact

This will limit the use of the MQTT servers, and it will take time to roll out and sort out the new settings. It gives users more direct control over their data.

## Protocol Buffer Changes

We would add a repeated field that contains a 16 bit latitude, longitude and an 8 bit precision field. This may go into ModuleSettings, for a per-channel config.
Additionally, there would be at least one setting for DoNotMqtt. This could be split into two, one for position, the other for all other messages. This may be a per-channel setting as well.

## Technical Details

For the home zones, the user would pick a position, and set a precision. The precision value would then be used to reduce the precision of the picked position, and the values stored in this reduces precision state.
When sending a position packet, the current position would be temporarily be precision reduced by each of the home zones precision setting, and the result compared to the stored position. If they match, 
the reduced precision value would be used.

It would be straightforward to check all messages the local node can decode, and simply refuse to send them on to MQTT if the bool is set true.


### Compatibility Considerations

The home zones are strictly a locla fimrware change, so no compatibility issues.
The doNotMqtt bit will be ignored by incompatible firmwares, but this should not affect deliverability. The MQTT server itself should be able to check for the byte on packets with known encryption.

### Security Considerations

The DoNotMqtt bit is intended to be binding on a policy level, but it's not technically binding. It would be trivial to compile a firmware that strips this bit on incoming messages.
It would be important to warn users that it is not a technically binding feature. On the other hand, it would be easy to test whether a firmware or MQTT server is honoring the bit. 
It would be possible, then, to warn users that they're attempting to connect to a server that has displayed malicious behavior in ignoring user wishes.

### Performance Considerations

The extra bit on all messages is minimal. The storage for settings isn't too terrible. The processing time should be minimal.

## Drawbacks

DoNotMqtt will reduce MQTT use, particualrly if we set it true by default. But that's probably exactly what we want. Showing up on a public map should be opt in, not opt out.

## Rationale and Alternatives

So far, these are the least heavy-handed solutions we've identified. 

## Prior Art

Strava uses a similar privacy zone system.

## Unresolved Questions

Do we put all these optins in the channel module configs? Is DoNotMqtt actually helpful? What exactly do we do about rogue, public MQTT servers that ignore the bit?
