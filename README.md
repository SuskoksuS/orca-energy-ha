## Home Assistant integration with Orca Energy 
Home Assistant Integration for Orca Energy Heat Pump. It supports models Mono and Duo with various configurations (floor, radiators, solar, DHW).


![alt text](dashboards/en-example.png?raw=true "")

After installing and configuring the integration, [follow these instructions to import the dashboard](dashboards/)

## Requirements
Orca Heat Pump should be accessible via network by HA. First test your access by connecting to http://{ip_addr_of_orca_hp}. Depending on firmware version, you might see either password prompt or just a generic landing page.

## Installation
1. Add Integration  
    A.  When using HACS add this repository as custom repository by opening HACS, select Integrations, press the three dots in the upper right corner, select Custom Repositories and add https://github.com/Tomasinjo/orca-energy-ha as new repository. Install it by pressing button right button "Explore & Download Repositories", search for "orca" and select "Orca Engery Heat Pump".

    B.  Without HACS, copy folder orca-ha to your custom_components directory, which is usually located in /config directory.

2. Go to Settings -> Devices & Services -> + Add Integration -> Search for "Orca".
Configure integration using admin/admin, and domain name or IP address of Orca Heat Pump.

## Radiator circuit compatibility fix

### Background

Some Orca heat pump installations use multiple heating circuits with a **single shared room temperature sensor**.

A common setup is:

* Heating Circuit 1 → floor heating
* Heating Circuit 2 → radiator heating
* One shared room temperature sensor used by both circuits

In these systems the heat pump **does not expose a separate room temperature tag for the radiator circuit**. Instead, only the primary circuit reports the room temperature.

Example:

HC1 room temperature → valid value
HC2 room temperature → -9999

The value **-9999** is used by the Orca API to indicate that the sensor **does not exist for that circuit**, not that the circuit itself is inactive.

However, the radiator circuit still provides valid operational data such as:

* circulation pump state (`2_Vklop_C2`)
* desired room temperature (`2_Zahtevana_RF_MK_2`)
* mixing valve state (`2_Delovanje_MP2`)
* mixing valve opening
* heating mode

This means the circuit is fully operational even though some sensors are not available.

### Problem in the original implementation

The original integration required **all probe tags for a circuit to return valid values** in order to consider the circuit available.

Example logic:

if len([t for t in tags if t in results_map]) == len(tags):
self.available_circuits.append(circuit_id)

For radiator circuits this often fails because sensors like:

* `2_Temp_RF2`
* `2_Temp_VF2`

return the value **-9999**.

As a result:

* Heating Circuit 2 was incorrectly detected as unavailable
* All radiator sensors became unavailable in Home Assistant
* The radiator climate entity could not function correctly

### What was changed

Two changes were introduced to improve compatibility with these installations.

#### 1. Relaxed circuit detection for Heating Circuit 2

Instead of requiring all probe tags, the circuit is now considered present if **at least one valid operational tag is available**.

Example logic:

if circuit_id == 2:
if present_count >= 1:
self.available_circuits.append(circuit_id)

Probe tags were also adjusted to use signals that are typically present in radiator systems:

* `2_Vklop_C2`
* `2_Zahtevana_RF_MK_2`
* `2_Delovanje_MP2`

This ensures the radiator circuit is detected whenever it is actually configured and active.

#### 2. Shared room temperature fallback

Because the heat pump uses **one shared room temperature sensor for both heating circuits**, Heating Circuit 2 may not expose its own temperature tag.

A fallback was added so that HC2 can use the room temperature from HC1 when its own value is missing.

Example logic:

if val is None and self._circuit_id == 2 and "hc_room_temp_1" in self.coordinator.data:
return self.coordinator.data["hc_room_temp_1"].value

This reflects the real hardware configuration where **both heating circuits rely on the same room temperature measurement**.

### Result

With these changes:

* radiator circuits are correctly detected
* valid radiator sensors appear in Home Assistant
* sensors returning -9999 are safely ignored
* climate control works even when **both heating circuits share a single room temperature sensor**

This improves compatibility with real-world Orca heat pump installations where radiator circuits do not expose a full set of sensors through the API.
