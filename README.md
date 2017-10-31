# ath9k_ap_5ghz

This repository contains a patch for the fedora kernel repo so that the Atheros drivers ignores regulatory domain in EEPROM and allows it to be used as an ap with hostapd in the 5GHz band.

# Details:

While playing around with my Netgear WNDA3200 (https://wikidevi.com/wiki/Netgear_WNDA3200), I found that even though it is easily possible to use them as access points with hostapd in the 2.4GHz-bands, the same is not possible in the 5GHz-bands out of the box. This work is mostly bases on the work of the OpenWRT community and twisteroid ambassador on github (https://github.com/twisteroidambassador/arch-linux-ath-user-regd).
I basically only adapted the patches for Fedora 26 with kernel 4.13. 

## The starting point ist the following:

- make sure that the ath9k_htc kernel module is loaded and working
- make sure that the regulatory settings are properly configured for your location (in my case Germany, otherwise set them with "iw reg set DE"): "iw reg get"
```
phy#1
country DE: DFS-ETSI
	(2400 - 2483 @ 40), (N/A, 20), (N/A)
	(5150 - 5250 @ 80), (N/A, 20), (N/A), NO-OUTDOOR, AUTO-BW
	(5250 - 5350 @ 80), (N/A, 20), (0 ms), NO-OUTDOOR, DFS, AUTO-BW
	(5470 - 5725 @ 160), (N/A, 26), (0 ms), DFS
	(57000 - 66000 @ 2160), (N/A, 40), (N/A)
```
- then "iw phy1 info" should show:
```
Frequencies:
			* 2412 MHz [1] (20.0 dBm)
			* 2417 MHz [2] (20.0 dBm)
			* 2422 MHz [3] (20.0 dBm)
			* 2427 MHz [4] (20.0 dBm)
			* 2432 MHz [5] (20.0 dBm)
			* 2437 MHz [6] (20.0 dBm)
			* 2442 MHz [7] (20.0 dBm)
			* 2447 MHz [8] (20.0 dBm)
			* 2452 MHz [9] (20.0 dBm)
			* 2457 MHz [10] (20.0 dBm)
			* 2462 MHz [11] (20.0 dBm)
			* 2467 MHz [12] (20.0 dBm) (no IR)
			* 2472 MHz [13] (20.0 dBm) (no IR)
			* 2484 MHz [14] (disabled)
Frequencies:
			* 5180 MHz [36] (20.0 dBm) (no IR)
			* 5200 MHz [40] (20.0 dBm) (no IR)
			* 5220 MHz [44] (20.0 dBm) (no IR)
			* 5240 MHz [48] (20.0 dBm) (no IR)
			* 5260 MHz [52] (20.0 dBm) (no IR, radar detection)
			* 5280 MHz [56] (20.0 dBm) (no IR, radar detection)
			* 5300 MHz [60] (20.0 dBm) (no IR, radar detection)
			* 5320 MHz [64] (20.0 dBm) (no IR, radar detection)
			* 5500 MHz [100] (disabled)
			* 5520 MHz [104] (disabled)
			* 5540 MHz [108] (disabled)
			* 5560 MHz [112] (disabled)
			* 5580 MHz [116] (disabled)
			* 5600 MHz [120] (disabled)
			* 5620 MHz [124] (disabled)
			* 5640 MHz [128] (disabled)
			* 5660 MHz [132] (disabled)
			* 5680 MHz [136] (disabled)
			* 5700 MHz [140] (disabled)
			* 5745 MHz [149] (disabled)
			* 5765 MHz [153] (disabled)
			* 5785 MHz [157] (disabled)
			* 5805 MHz [161] (disabled)
			* 5825 MHz [165] (disabled)
```

- The big problem is now, that even though communication is allowed in 8 5GHz-channels, the setting in the EEPROM of the wireless adapter forbid it. 
- My device has the regulatory domain 0x65 set ("dmesg | grep ath" gives ath: EEPROM regdomain: 0x65). According to https://wireless.wiki.kernel.org/en/users/drivers/ath this is one of 12 custom regulatory domains that Atheros uses. The 5GHz channels are marked with the "no IR"-flag, which means that the device is not allowed to initiate transmissions in theses bands. It has to wait for an access point to send a beacon and only upon receivinf this beacon it can start to transmit in the 5GHz band. 
- This behavior is artificial and not necessary to adhere to regulatory rules. One might suspect that this was configured this way in order to be able for a company to charge more for "special" wireless adapters for access points.

## What to do: (instructions for Fedora)

- first make sure that you have all dependencies to build a custom kernel (https://fedoraproject.org/wiki/Building_a_custom_kernel), then do
```
fedpkg clone -a kernel
git branch f26 origin/f26
git checkout f26
make release
```
- then apply the patches:  ath_Kconfig.patch  ath_regd.patch reg.patch
 
- build the kernel
```
fedpkg local
dnf install --nogpgcheck ./x86_64/kernel-$version.rpm
```

- after rebooting into the new kernel you should get:
```
		Frequencies:
			* 5180 MHz [36] (20.0 dBm)
			* 5200 MHz [40] (20.0 dBm)
			* 5220 MHz [44] (20.0 dBm)
			* 5240 MHz [48] (20.0 dBm)
			* 5260 MHz [52] (20.0 dBm) (radar detection)
			* 5280 MHz [56] (20.0 dBm) (radar detection)
			* 5300 MHz [60] (20.0 dBm) (radar detection)
			* 5320 MHz [64] (20.0 dBm) (radar detection)
```
## Settings for Hostapd:

```
hw_mode=a
channel=36
ssid=bla
#wme_enabled=1
ieee80211n=1
ieee80211d=1
ieee80211h=1
country_code=DE
```

