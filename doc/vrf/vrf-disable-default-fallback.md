# SONiC HLD: No Default VRF Fallback

## 1. Feature Name: No Default VRF Fallback

## 2. Goal:
To provide a configurable mechanism in SONiC to disable fallback to the default VRF when a route is not found in a non-default VRF. This gives network operators more control over routing behavior and potentially improves security.

## 3. Overview:

The solution introduces a configurable option to prevent the system from falling back to the default VRF when a route lookup fails in a non-default VRF. This is achieved by adding unreachable default routes (IPv4 and IPv6) to the Linux VRF tables associated with the non-default VRFs. The configuration can be applied globally or overridden on a per-VRF basis. No kernel changes are required.

## 4. Key Components and Interactions:

*   **CLI (sonic-utilities):** Provides commands to configure the global and per-VRF fallback settings.
*   **CONFIG\_DB:** Stores the global and per-VRF fallback configurations.
*   **VRF Manager Daemon (vrfmgrd in sonic-swss):** Monitors the CONFIG\_DB for changes to the fallback settings. Based on the configuration, it adds or removes unreachable default routes in the appropriate Linux VRF tables.
*   **Linux Kernel:** The underlying routing infrastructure. The solution manipulates the VRF routing tables by adding/removing unreachable routes.

## 5. Detailed Design:

*   **Configuration:**
    *   **Global Setting:** A global setting to enable or disable the no-default-VRF-fallback behavior. The default is *disabled* (i.e., fallback *is* allowed, which is the current SONiC behavior).
    *   **Per-VRF Setting:** An override setting for each VRF to enable or disable the no-default-VRF-fallback behavior. This setting takes precedence over the global setting.
*   **Data Model (CONFIG\_DB):**
    *   A new table `NO_DEFAULT_FALLBACK_GLOBAL` stores the global setting.

        ```
        "NO_DEFAULT_FALLBACK_GLOBAL|settings": {
            "status": "enabled"  // or "disabled"
        }
        ```
    *   A new field `fallback_to_default_vrf` is added to the `VRF` table.

        ```
        "VRF|VrfRed": {
            "fallback_to_default_vrf": "false" // or "true"
        }
        ```
*   **CLI Commands:**
    *   Global Configuration:
        *   `config vrf no-default-fallback global enable`
        *   `config vrf no-default-fallback global disable`
    *   Per-VRF Configuration:
        *   `config vrf no-default-fallback vrf enable <VRF_NAME>`
        *   `config vrf no-default-fallback vrf disable <VRF_NAME>`
    *   Status Display:
        *   `show vrf no-default-fallback`
*   **VRF Manager Daemon (vrfmgrd):**
    1.  Monitors the `NO_DEFAULT_FALLBACK_GLOBAL` table and the `VRF` table in CONFIG\_DB.
    2.  For each VRF:
        *   Determines whether fallback should be disabled:
            *   If `VRF[VRF_NAME]['fallback_to_default_vrf']` exists, use its value. `"false"` disables fallback; `"true"` enables it.
            *   Otherwise, use the global setting from `NO_DEFAULT_FALLBACK_GLOBAL['settings']['status']`. `"enabled"` disables fallback; `"disabled"` enables it.
        *   If fallback is disabled:
            *   Adds an unreachable IPv4 default route to the VRF's routing table: `ip route add unreachable default metric 4278198272 vrf <VRF_NAME>`
            *   Adds an unreachable IPv6 default route to the VRF's routing table: `ip -6 route add unreachable default metric 4278198272 vrf <VRF_NAME>`
        *   If fallback is enabled:
            *   Removes the unreachable IPv4 default route from the VRF's routing table: `ip route del unreachable default vrf <VRF_NAME>`
            *   Removes the unreachable IPv6 default route from the VRF's routing table: `ip -6 route del unreachable default vrf <VRF_NAME>`
*   **Config Reload:**
    *   The `vrfmgrd` must reconcile the routes on startup to ensure the configured fallback behavior is enforced after a config reload. This involves reading the configuration from CONFIG\_DB and adding/removing the unreachable routes as described above.

## 6. Error Handling:

*   The CLI should validate the VRF name before updating the CONFIG\_DB.
*   The `vrfmgrd` should log errors if it fails to add or remove the unreachable routes.

## 7. Scalability and Performance:

*   The impact on performance should be minimal. Adding/removing a few unreachable routes should not significantly affect routing performance.
*   The solution should scale to support a large number of VRFs.

## 8. Testing:

*   **Unit Tests:**
    *   CLI command parsing and validation.
    *   CONFIG\_DB schema validation.
    *   VRF Manager Daemon logic.
*   **Integration Tests:**
    *   Verify default route fallback behavior with and without the unreachable route.
    *   Verify per-VRF overrides.
    *   Verify global setting application.
    *   Verify that the configuration persists across config reloads.
    *   Test IPv4 and IPv6 scenarios.

## 9. Alternatives Considered:

*   The document mentions that kernel changes were avoided. This was likely a major consideration, as kernel changes are more complex and require more extensive testing.

## 10. Open Issues:

*   None mentioned in the document.