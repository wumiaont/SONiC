# SONIC FIPS Poser On Self Test HLD

## Revision

|  Rev  | Date       | Author     | Change Description |
| :---: | :--------: | :--------: | ------------------ |
|  0.1  | 2025-06-01 | Wu Miao    | Initial version    |


## Table of Contents
- [Abbreviation](#abbreviation)
- [Requirement](#requirement)
- [The cryptographic modules in SONiC](#the-cryptographic-modules-in-SONiC)
- [OpenSSL FIPS 140-3](#OpenSSL-FIPS-140-3)
  * [OpenSSL Engine](#OpenSSL-Engine)
  * [SymCrypt OpenSSL Engine](#symCrypt-openSSL-engine)
  * [OpenSSL configuration for SymCrypt Engine](#OpenSSL-configuration-for-SymCrypt-Engine)
  * [OpenSSL configuration enhancement](#OpenSSL-configuration-enhancement)
  * [SymCrypt OpenSSL Engine debian package](#SymCrypt-OpenSSL-Engine-debian-package)
- [Kerberos Cryptographic Module](#Kerberos-Cryptographic-Module)
- [Golang Cryptographic Module](#Golang-Cryptographic-Module)
- [Application Impact](#Application-Impact)
- [SONiC FIPS Configuration](#SONiC-FIPS-Configuration)
  * [Enable FIPS on system level](#Enable-FIPS-on-system-level)
  * [Enable FIPS on application level](#Enable-FIPS-on-application-level)
  * [SONiC Build Options](#SONiC-Build-Options)
- [SONiC FIPS Command lines](#SONiC-FIPS-Command-lines)
- [Q&A](#Q&A)


## Abbreviation

| Abbreviation | Description                                  |
| ------------ | -------------------------------------------- |
| CAVP         | Cryptographic Algorithm Validation Program   |
| CMVP         | Cryptographic Module Validation Program      |
| FIPS         | Federal Information Processing Standard      |
| POST         | Power On Self Test                           |

## Introduction
SONiC only uses cryptographic modules validated by FIPS 140-3, Make SONiC compliant with FIPS 140-3 when FIPS is enabled.

This document describes the design of validating the hardware (Switch ASIC) to be FIPS 140-3 compliant for MACSEC and IPSEC.

FIPS 140-3 compliance requires that POST is executed on each MACSec/IPSec port and only if POST completes with ‘success’, the MACSec/IPSec port can be enabled to forward traffic.

OpenComputerProjects introduced new SAI APIs for triggering POST for FIPS 140-3 standard compliance to overall security level 1.

## New SAI APIs to Support POST Test
We will only list SAI APIs which SONIC will use for its MACSEC POST test.

New attribute is introduced to trigger POST at the SAI MACSec engine id. Setting the attribute to true will start POST on all the physical MACSec engines and ports associated with the MACSec engine id.

    /**
     * @brief Setting the value to true will start the post on all the ports serviced by this MACSEC engine
     *
     * @type bool
     * @flags CREATE_ONLY
     * @default false
     */
    SAI_MACSEC_ATTR_ENABLE_POST
Status of the POST can be queried using SAI_MACSEC_ATTR_POST_STATUS attribute. Note that status reflects the aggregate status of all the engines and ports serviced by this MACSec object id. Even if a single MACSec port fails the POST, status of the MACSec object will be returned as SAI_MACSEC_POST_STATUS_FAIL. Subsequently NOS has to query individual MACSec engines/ports serviced by the MACSec object id to figure out which MACSec engine/port has failed the POST.

    /**
     * @brief MACSEC POST status
     * Attribute to query the status of POST for a MACSEC engine
     *
     * @type sai_macsec_post_status_t
     * @flags READ_ONLY
     */
    SAI_MACSEC_ATTR_POST_STATUS
Following POST status values are defined. SAI_MACSEC_POST_STATUS_IN_PROGRESS means that POST is still running and is not yet complete.

    /**
     * @brief Attribute data for #SAI_MACSEC_ATTR_POST_STATUS,
     */
    typedef enum _sai_macsec_post_status_t
    {
        /** Unknown */
        SAI_MACSEC_POST_STATUS_UNKNOWN,
    
        /** Pass */
        SAI_MACSEC_POST_STATUS_PASS,
    
        /** In Progress */
        SAI_MACSEC_POST_STATUS_IN_PROGRESS,
    
        /** Fail */
        SAI_MACSEC_POST_STATUS_FAIL,
    } sai_macsec_post_status_t

Single aggregate callback function is provided to return the status of POST status for the entire MACSec engine.

If the engine is servicing a single port then this callback essentially becomes a per port callback.

If the engine is servicing multiple ports then each port needs to be queried for its POST status using the READ only attribute of the port.

    /**
     * @brief MACSEC post status notification
     *
     * @objects switch_id SAI_OBJECT_TYPE_MACSEC
     *
     * @param[in] macsec_id MACSEC Id
     * @param[in] macsec_post_status MACSEC post status
     */
    typedef void (*sai_macsec_post_status_notification_fn)(
            _In_ sai_object_id_t macsec_id,
            _In_ sai_macsec_post_status_t macsec_post_status);
    
      /**
       * @brief Attribute Id in sai_set_switch_attribute() and
       * sai_get_switch_attribute() calls.
       */
      typedef enum _sai_switch_attr_t
      {
          /**
           * @brief Callback for completion status of all the MACSEC ports serviced by this MACSEC engine
           *
           * Use sai_macsec_post_status_notification_fn as notification function.
           *
           * @type sai_pointer_t sai_macsec_post_status_notification_fn
           * @flags CREATE_AND_SET
           * @default NULL
           */
          SAI_SWITCH_ATTR_MACSEC_POST_STATUS_NOTIFY,
     }

Please check the following link for port level MACSEC corresponding IPSEC SAI APIs and more detailed MACSEC SAI APIs(https://github.com/opencomputeproject/SAI/blob/master/doc/fips/SAI-Proposal-FIPS-Compliance.md).


![FIPS Overview](images/fips-overview.png)

## SONIC Host Services Support for POST Test
SWSS needs to know the FIPS state of the chassis. Currently the FIPS information is saved only on host State database. In order for namespace based service such as SWSS to get FIPS information, FIPS config information needs to be populated to based database.

Following code needs to be added into sonic-host-services procDockerstats in update_fipsstats_command.
        # Update the FIPS state to STATE_DB in namespaces in multi-asic platform
        for ns, tbl in self.ns_fips_state_tbl.items():
            tbl.set("state", [('timestamp', datetime.utcnow().isoformat())])
            tbl.set("state", [('enforced', str(enforced))])
            tbl.set("state", [('enabled', str(enabled))])

## SONIC SWSS Support for POST Test
In SWSS orchagent main(), needs to check if fips is enabled for the sonic or not. If FIPS is anabled, needs to add following code into attrs list before calling switch_create. This will enable POST test for the macsec engine within the switch and also register a callback function to handle POST result.

        attr.id = SAI_SWITCH_ATTR_MACSEC_ENABLE_POST;
        attr.value.booldata = true;
        attrs.push_back(attr);
        SWSS_LOG_NOTICE("Enabled FIPS MACSEC POST test.");

        attr.id = SAI_SWITCH_ATTR_MACSEC_POST_STATUS_NOTIFY;
        attr.value.ptr = (void *)on_macsec_post_status;
        attrs.push_back(attr);

## SONIC Syncd Support for POST Test






### Enable FIPS on application level
```
export ENABLE_FIPS=1
```

Alternative option for the golang applications only:
```
export GOLANG_FIPS=1
```

Alternative option for the OpenSSL applications only:

see https://www.openssl.org/docs/manmaster/man7/openssl-env.html
```
export OPENSSL_CONFIG=/usr/lib/ssl/openssl-fips.cnf
```

### SONiC Build Options
Support to enable/disable the FIPS feature, the feature is enabled by default in rules/config as below.
```
INCLUDE_FIPS ?= y
```
Support to enable/disable FIPS config, the flage is disabled by default. IF the option is set, then the fips is enabled by default in the image, not necesary to do the config in system level or application level.
```
ENABLE_FIPS ?= n
```
If the INCLUDE_FIPS is not set, then the option ENABLE_FIPS is useless.

## SONiC FIPS Command lines
### The command line to enable or disable FIPS
sonic-installer set-fips <image> [--enable-fips|--disable-fips]

If the image is not specified, the next boot image will be used.
The default behavior is to enable FIPS, if none of the option --enable-fips or --disable-fips specified.

### The command line to show FIPS status
sonic-installer get-fips <image>

Returns the following message: FIPS is enabled/disabled.
If the image is not specified, the next boot image will be used.


## Q&A
### Does SymCrypt use Linux Kernel crypto module?
SymCrypt on Linux does not rely on Kernel crypt for FIPS certification today.
