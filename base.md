Syntax:
    #    = slide title
    |    = show in next step (i.e., press button to show)
    |[n] = show in step n (if | are out of order)
    ---  = slide delimiter
    The rest is just standard Markdown

---

# Sighax and Boot9strap

---

# A Short Summary

At 33c3, we learned the following:

* Boot9 has a broken RSA verification implementation.
* It's broken because the ASN.1 parser incorrectly verifies at least one of the length fields.
* We can't perform sighax ourselves, because we can't get the details required from Boot9's parser, because we don't have Boot9

|What can we do with that?

---

# A Blast from the Past

* Nintendo loves reusing concepts and code. Firmware has to parse RSA signatures, too.
* Initial examinations of NATIVE_FIRM, TWL_FIRM and AGB_FIRM look not vulnerable -- though the ASN.1 parser looks very similar to what was described at 33c3, there are stringent padding checks in place.
* Checks against the sighax vulnerability are present in firmware as early as 0.11, from July 2011.
|* For some reason, **they took it out on 0.13, which is the factory firmware version**.
|* Factory firmware is vulnerable to sighax.
|* The checks were re-added in firmware 1.0.0-0.

---

![picture of sighax-vulnerable ASN.1 parser in Hex-Rays](...)

|![wat](https://i.imgur.com/SwfNTmx.jpg)

---

# Exploiting a parser, blindly

* The parser in factory firmware isn't exactly the same as the one in boot9, as learnedwhen a "perfect" signature for factory firmware's parser failed on boot9. What's next?
|* Testing a signature that overflows the factory firmware parser into data-aborting causes the console to show a black screen instead of bootrom error. This means we can crash the bootrom.
|* On factory firmware, the "calculated" hash is on the stack near the signature being parsed. This is probably the case for boot9.
|* There are only 128 possible locations, relative to the signature, that can be brute forced in reasonable time. It has to be one of those.
|* Sure enough, brute forcing signatures for each location shows the "calculated" hash is immediately after the signature on the stack. So we have sighax.

---

# So we have sighax. Now What?

**We dump the bootroms, of course!**

---

# Exfiltrating Boot9

* So we can fakesign our own FIRMs.
* Meaning we can copy our payload almost anywhere.
* Boot9 has a blacklist of ranges... but we know it's not perfect.
* From 33c3, only the boot9 data regions are blacklisted.
|* Can we find a region that isn't a boot9 data region but is still dangerous?

---

# ARM9 Memory Map

(steal table on 3dbrew Memory_layout Boot9 section)

|(highlight region 1, which is the I/O register region)

---

# NDMA

* We can copy anything almost anywhere.
* This includes I/O registers.

|(screenshot of 3dbrew "NDMA can access the (unprotected) part of the bootrom.")
|(hide screenshot)

|* The NDMA engine is an I/O register. This means we can copy data over the NDMA registers, triggering DMA, letting us copy anywhere.

---

# NDMA Parameters

For an NDMA copy request, we need:

* To set the global control register to `high priority`.
* a source address (`0xFFFF8000`, the address of the protected Boot9)
* a destination address (Any safe place in arm9mem will do)
* how much data to transfer (32KiB, or 8192 words)
* some timing data
* how much to transfer per cycle
|* And now we just copy that to `0x10002000` for a copy request!

---

# Works for me, too

SHA-512 (Boot9):

5addf30ecb46c624fa97bf1e83303ce5  
d36fabb4525c08c966cbaf0a397fe4c3  
3d8a4ad17fd0f4f509f547947779e1b2  
db32abc0471e785cacb3e2dde9b8c492

---

# But wait, there's more!

![Billy Mays](https://upload.wikimedia.org/wikipedia/commons/f/fa/Billy_Mays_Portrait_Cropped.jpg)

---

# Boot9strap: Reliable Boot9/Boot11 Code Execution

* We can copy data anywhere during firm loading.
* Including:
    |* the exception vectors (using NDMA to bypass the blacklist), and
    |* NULL.
|* Oh. That's all we need for a data abort, isn't it?
|* **It is!**

---

# boot9strap: Big picture idea

* Section 0: Copy an ARM11 payload into axiwram
|* Section 1: Copy an ARM9 payload into arm9mem
|* Section 2: Copy a payload over the NDMA registers that overwrites the data abort vector in arm9mem.
|* Section 3: Load anything to NULL.
|* ???
|* Profit!

---

# boot9strap: Technical implementation

* Getting execution during boot9 via data abort is really cool. But it'd also be nice to get execution under boot11. How can we?
|* Boot9 dereferences some function pointers from DTCM, and calls them if they aren't NULL (they normally are).
|* Our poisoned data abort vector can overwrite two of these to point at our code, then return to boot9 and let it resume normally.
|* Later on, boot9 will call the first of those function pointers (`0xFFF00058`). This will wait for boot11 to finish a task b by watching axiwram, then overwrite a function pointer (`0x1FFE802C`) it will call later.
|* Immediately before lockout, boot9/boot11 will each dereference function pointers we have poisoned (`0xFFF0005C`, `0x1FFE802C` respectively).
|* We can execute arbitrary code before bootrom lockout under both boot9 and boot11.

---

# Works for me, 2: Electric Boogaloo

SHA-512 (Boot11):

c3f5044321f1ec5a33e09dd021e1117b  
1b663e37673222944a06b119ea70af92  
fc6de8547129582131fb9f8896edef7e  
9a8ef77295c627e0f1cb360b15b533b4

---

# Bonus fail

* Upon disassembling boot9, we notice another huge flaw in the bootrom that wasn't mentioned at 33c3.
|* Before trying to boot from NAND, the bootrom checks to see if a key combination (Start + Select + X) is being held.
|* If so, it tries to boot from an inserted NTR (Nintendo DS) cartridge.
|* Combined with sighax/boot9strap, this allows one to make a malicious fake DS cartridge, so that holding down a button combination on boot gives you bootrom code execution.
|* This, like sighax, is also not fixable.
|* The NTR cartridge was likely meant to be used for either the factory setup or as a means of recovering bricked NANDs. However, we'll never know for sure.

---

![vitcory shibe doge](https://i.imgur.com/TKiEXfp.png)

@SciresM, @Myriachan, [Normmatt](https://github.com/Normmatt/)

