# Chat2Shell
Potential Remote Code Execution Vulnerability in Minecraft Mods

**Status: Unpatched / Under Investigation 
Severity: Critical (RCE)
Warning: This vulnerability is still under active investigation. The exact root cause and affected mod versions have not yet been independently verified, still do not call this false information since we tested it on our own device and provided work.**

Chat2Shell is a potential security vulnerability affecting Minecraft mods that read, parse, or store incoming in game chat messages using unsafe text handling or non UTF-8 implementations. Under specific conditions, a specially crafted chat message using non-standard text encodings (such as UTF-16 or malformed SQLite text payloads) can overload downstream processing on the client, potentially leading to local command execution.

In preliminary testing, the behavior was reproduced under the following setup:
• OS & Architecture: Windows 10 or newer (AMD64 / x86-64).
• Hardware Context: Tested on a system with 8 GB RAM; CPU speed and memory appear to affect timing and reliability.
• Mod Setup: Both the server and client must run an affected version of a compatible chat-reading or logging mod.

## Technical Background
The flaw lies in mod level chat parsing or reading. While standard Minecraft chat uses UTF-8-compatible serialization, the tested payload abuses non-UTF-8/UTF-16 Symbols representations passed into SQLite or native text parsers. During testing, unexpected local process execution occurred roughly 10 seconds after the message was received, suggesting asynchronous queue flushing or buffer handling issues.

## Exploit Mechanism & PoC Policy
For security reasons, no Code or functional exploit payload will be released until affected mods are patched or more information gotten At a high level, the attack works as follows: 

1. Crafted Input: An attacker sends a chat message containing compressed Java code interspersed with UTF-16 symbol blocks to overload standard text parsing.
2. Overload: When a mod with outdated chat reading or logging logic processes the message, the mixed representation causes log storing to stall or overload, triggering a DoS state.
3. Runtime Execution: Under this overloaded state, the client's chat handling path running inside Minecraft's own Java Runtime Environment allegedly fails to cleanly reject or isolate the input and instead passes it toward an execution sink, resulting in unintended local execution.
_Additionally, there may be an underlying edge case in Java’s binary chat stream handling itself: normally, malformed binary chat payloads are filtered or trigger a clean client crash before causing further harm, whereas in this case the check appears to have been **somehow** bypassed resulting in a DoS condition and subsequent execution for reasons that remain unconfirmed._

Example Chat Input: R□u□n□t□i□m□e□.□g□e□t□R□u□n□t□i□m□e□(□)□e□x□e□c□(□“□c□a□l□c□.□e□x□e□)□;□
If the attacker used that example just with more than 4096 Bytes and a Client used outdated chat reading and stores that logs into a UTF-8 Database, it causes to overload.

## Discovery & Authenticity
This finding is not a fake report or recycled public exploit, though independent verification is still ongoing. It was first noticed during live play on a Minecraft server when myanti-exploit/anti-cheat check flagged unusual activity on a mod. Decompiling the mod revealed outdated Java chat reading routines that interact poorly with malformed text and overload behavior, leading to further analysis of how a DoS condition and legacy functions can chain into active RCE. Because reverse engineering and testing were done privately by a small research group, full independent confirmation and patch testing are still in progress.
Even thought Vanilla Minecraft strictly limits standard chat packets to 256 characters on both the client and server. These Vanilla server wont be affected in any case, but modified server that explicitly bypasses packet size limits and allow UTF-16 Symbols could be abused.

## Examples of Mod Architectures to Audit
Any mod that intercepts, logs, or synchronizes chat packets between client and server—especially those writing chat history to local SQLite databases—should be reviewed. Examples of mods that implement client/server chat reading, formatting, or SQLite logging patterns include:
• Client/Server Chat Processors: Mods like Beautified Chat (which has paired client and server variants for parsing and formatting chat).
• UTF-16 Chat & Event Loggers: Mods that store chat or server logs in databases, such as Chat Logs experimental mode, ChatTracker, or server/client loggers like Tempora and GriefLogger.
_Additionally, there may be an underlying edge case in Java’s binary chat stream handling itself: normally, malformed binary chat payloads are filtered or trigger a clean client crash before causing further harm, whereas in this case the check appears to have been **somehow** bypassed resulting in a DoS condition and subsequent execution for reasons that remain unconfirmed._

**(Note: Listing these mods does not mean they are confirmed vulnerable, they represent the type of chat reading and UTF16 logging functionality relevant to this investigation.)**

## Mitigations & Temporary Precautions
No permanent patch is available yet. Until maintainers verify and patch affected codebases:
• Disable or Remove: Disable chat-reading/logging features in potentially affected mods, or temporarily remove the mods from both client and server.
• Enforce Strict UTF-8: Reject incoming chat messages containing invalid UTF-8 byte sequences, raw UTF-16 surrogates, or unexpected control characters before they reach mod parsers or non UTF 8 symbols backends. In initial testing, strict UTF-8 validation blocked the trigger vector, though it should only be treated as a temporary defense in depth measure.
• Sanitize Database Queries: Ensure any chat storage uses parameterized queries (PreparedStatement) and validated character encodings.

## Investigation Status
• Initial testing and reverse engineering completed
• Initial security report drafted
• Root cause and execution sink independently verified
• Confirmed list of affected mods published
• Permanent patches released and verified

## Responsible Disclosure & Disclaimer
This advisory is shared strictly for defensive awareness and responsible disclosure. No weaponized payloads are included. Do not use this information to test unauthorized systems or disrupt players. If you maintain a mod that processes or logs chat and want to audit your code, please contact us privately.

## FAQ
**Why is there no Proof of Concept code in this repository?**
This vulnerability does not rely on a standalone script or external tool that can simply be uploaded and run. The trigger requires a specific combination of live server side packet manipulation (bypassing standard Vanilla size limits), specific clien mod configurations, and malformed UTF-16 byte sequences. Releasing the exact raw payload right now would immediately weaponize the bug against unpatched servers, which goes against responsible disclosure practices.

**How can a simple chat message or database entry lead to Remote Code Execution ?**
Storing corrupted text in a database does not execute Java code by itself. However, mods that actively parse, format, or synchronize chat packets pass these strings through legacy Java text-handling routines and native serialization layers. Under heavy buffer overload caused by oversized (>4096 bytes) mixed encoding payloads, flawed error handling fails to isolate the input, allowing the data to reach an active execution sink within the Minecraft Java Runtime.

**People are calling this fake because they haven't seen it reproduced yet. How do we know it's real?**
Independent verification takes time, especially when dealing with timing sensitive memory issues that depend on JVM behavior and specific hardware states. Skepticism is completely normal in security research until a CVE or official patch is published. We are currently working on a safe, non weaponized reproduction environment (a dummy mod) so maintainers can verify the crash and execution path locally without exposing real servers to risk.

**What should mod developers or server owners do right now?**
Even if you remain skeptical, enforcing strict UTF-8 validation and rejecting oversized chat packets (>256 characters) is basic defensive hygiene. If you maintain a mod that logs or modifies chat, audit your PreparedStatement usage and ensure malformed byte sequences are dropped immediately before hitting your parsing logic.
