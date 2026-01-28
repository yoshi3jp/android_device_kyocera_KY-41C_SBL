![TWRP](https://raw.githubusercontent.com/TeamWin/twrpme/refs/heads/master/favicon.ico "for TWRP")
# Android device tree for KYOCERA KY-41C (KY-41C)

## Contributors
 -	[Yoshinobu Date](https://github.com/yoshi3jp) 

```
#
# Copyright (C) 2026 The Android Open Source Project
# Copyright (C) 2026 SebaUbuntu's TWRP device tree generator
#
# SPDX-License-Identifier: Apache-2.0
#
``` 
## Decryption library trick
There was an issue where libkeymaster_portable.so had an older nearly yet not quite compatible implementation of EcKeyFactory::CreateEmptyKey, causing libsoftkeymasterdevice.so to fail. As a fix, a compatibility layer of shared library was added as a potentially viable option for recovery-only use case. If you adopt this tree for different kind of project, it may not work or perform well. On the otherhand this trick can potentially work on other device so long as the objective is the same. (TWRP 12.1 for Android 10 MTK-ARMv7-neon-Trustonic device)

---

## HAL Debugging Notes (lshal, debugfs, recovery)

This device tree includes **explicit support for HAL introspection using `lshal`**.
This section documents **why this was necessary**, **what was required to make it work in recovery**, and **recommended defaults for future debugging**.

These notes are particularly relevant when dealing with **legacy vendor HALs (HIDL, API ≤29)** under **Android 12+ recovery environments (TWRP / custom recovery)**.

---

### 1. Why `lshal` was added

When debugging crypto, Gatekeeper, Keymaster, Trustonic TEE, or other vendor HALs, it is often insufficient to rely on:

* init logs alone
* service startup success
* or “it exists under `/vendor/bin/hw/`”

**We need to know:**

* Which HAL services are actually registered with `hwservicemanager`
* Whether they are **binderized vs passthrough**
* Which processes are **clients** of those HALs
* Whether recovery itself is calling them

`lshal` is the only practical tool that provides this **binder-level topology** in a structured way.

Without `lshal`, it is very easy to misdiagnose:

* “HAL missing” vs
* “HAL present but unused” vs
* “HAL present but unreachable due to policy / binder / compatibility issues”

---

### 2. What was required to make `lshal` usable in recovery

Simply adding the `lshal` binary is **not sufficient**.
Out of the box, `lshal` in recovery will often print misleading warnings such as:

```
Warning: Skipping "...": no information for PID XXX, are you root?
```

This is **not** caused by lack of root privileges.

#### Root cause: missing binder debugfs

`lshal` relies on **binder debug information** exposed via **debugfs** to compute:

* Thread usage
* Client PIDs
* Binder relationships (who talks to whom)

If debugfs is not mounted, `lshal` can still list services, but:

* `Thread Use` will be empty or incorrect
* `Clients` will be missing
* misleading “are you root?” warnings appear

#### Required conditions

To obtain **full, reliable `lshal` output**, the following must be true:

1. **debugfs must be mounted**

   * Path: `/sys/kernel/debug`
   * Symlink `/d → /sys/kernel/debug` is not enough by itself

2. **Binder debug nodes must be present**

   * `/d/binder/state`
   * `/d/binder/transactions`
   * `/d/binder/stats`
   * `/d/binder/transaction_log`

3. Recovery shell must have:

   * access to `/proc` (readable PIDs)
   * access to `/sys/kernel/debug`
   * working binder devices:

     * `/dev/binder`
     * `/dev/vndbinder`
     * `/dev/hwbinder`

Once debugfs is mounted, `lshal` immediately begins reporting:

* accurate client lists
* thread usage
* correct binder topology

This was verified empirically on this device.

---

### 3. Recommended: mount debugfs during recovery init

To avoid confusion and repeated setup for every debugging session, **debugfs should be mounted early during recovery init**.

#### Recommended init action

Add the following to a recovery `.rc` file (or equivalent):

```rc
on early-init
    mkdir /sys/kernel/debug 0755 root root
    mount debugfs debugfs /sys/kernel/debug
```

This ensures:

* `/d/binder/*` is available immediately
* `lshal` works out of the box
* HAL debugging does not depend on manual intervention
* developers can reliably verify:

  * HAL presence
  * binder registration
  * client relationships

#### Notes

* This assumes the kernel was built with `CONFIG_DEBUG_FS=y`
* On production builds this may be undesirable, but **for recovery / development builds it is strongly recommended**
* SELinux may log permissive AVCs in recovery; this is expected and acceptable for debugging

---

### 4. Practical impact for HAL / crypto debugging

With `lshal` + debugfs enabled, we were able to confirm that:

* Gatekeeper, Keymaster 3.0, Trustonic TEE, and vendor attestation HALs **were present and running**
* They were **registered with hwservicemanager**
* **Recovery itself was not a client** of these HALs (only `hwservicemanager` was)

This distinction is critical:

* The problem was **not missing HALs**
* The problem was **plumbing / compatibility / usage**, not availability

Without `lshal` + debugfs, this conclusion would have been impossible to reach with confidence.

---

### 5. Summary

**For anyone working on this device tree or similar ports:**

* `lshal` is a required debugging tool
* debugfs **must** be mounted for meaningful output
* misleading `lshal` warnings almost always mean “no debugfs”, not “no root”
* enabling this early saves significant time when dealing with:

  * keystore2 vs legacy keymaster
  * fscrypt / vold issues
  * Trustonic / TEE integration
  * HIDL ↔ AIDL transition problems

Keeping this infrastructure in place makes HAL-level debugging **tractable instead of guesswork**.
