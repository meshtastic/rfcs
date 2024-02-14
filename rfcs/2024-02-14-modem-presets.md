_Using the format from @Tekstrand, as regular template is missing_

## RFC

This Request for Comments (RFC) seeks to address a recurring concern within the Meshtastic community regarding the the existing Meshtastic preset names, for instance "LongFast". Meshtastic employs modem presets in order to allow users to change the behavior of their LoRa radios, affecting maximum potential range, latency, and bandwidth. The naming convention for these modem presets is currently a carryover from Meshtastic "1.0". Most critically, the presets represent a continuum of bandwidth options, rather than strictly "Fast" or "Slow" effects.
### Problem Statement

 The words "Long", "Medium", "Short", "Fast", and "Slow" used in the modem presets do not represent either theoretical or real world implications/effects on LoRa radio peformance. The terms used for the Mesthastic modem presets, such as "LongFast", are misleading. The terminology such as "Fast", "Slow", "Long", and "Short" cause users to assume that each preset has tradeoffs. For example users may assume "MediumFast" is only  for moderate (which is not defined or tested) ranges, while also assuming the Slow option will result in slow message sending or some other vague effects.  

The current language complicates user interaction, as there is little reason a user may want maximum range in all scenarios, without understanding that the change in presets may adversely affect the radio's performance, particularly on "LongSlow".

### Objective

The primary objective of this RFC is to **initiate a community-driven discussion** aimed at reevaluating the term "channel." By addressing this terminology concern, we aim to enhance clarity, reduce confusion, and improve the overall user experience on the Meshtastic network.
### Considerations

    1. Terminological Clarity: Any new term should unambiguously describe its intended concept without overlap with existing radio or communication terms.

    2. User Familiarity: The replacement should leverage terminology familiar to both new and existing users, facilitating ease of adoption.

    3. Technical Accuracy: The term should accurately reflect the technical nature of Meshtastic's features and functionalities.

    4. Community Feedback: Emphasizing a collaborative approach, this RFC encourages feedback and suggestions from the Meshtastic community.


### Call to Action
+
We invite all community members to contribute their perspectives, suggestions, and insights regarding the problematic use of "channel". This collaborative effort will ensure that any terminology changes will serve to enhance understanding, usability, and community engagement with Meshtastic.

This RFC represents an opportunity to refine our language and align more closely with our platform's innovative spirit. Your participation is crucial to achieving a consensus that reflects the diverse experiences and needs of the Meshtastic community.
