# Chat2Shell
Potential Remote Code Execution Vulnerability in Minecraft Mods

**Status: Unpatched / Under Investigation | Severity: Critical (Potential RCE)
Warning: This vulnerability is still under active investigation. The exact root cause and affected mod versions have not yet been independently verified.**

Chat2Shell is a potential security vulnerability affecting Minecraft mods that read, parse, or store incoming in game chat messages using unsafe text handling or non UTF-8 implementations. Under specific conditions, a specially crafted chat message using non-standard text encodings (such as UTF-16 or malformed SQLite text payloads) can overload downstream processing on the client, potentially leading to local command execution.

In preliminary testing, the behavior was reproduced under the following setup:
• OS & Architecture: Windows 10 or newer (AMD64 / x86-64).
• Hardware Context: Tested on a system with 8 GB RAM; CPU speed and memory appear to affect timing and reliability.
• Mod Setup: Both the server and client must run an affected version of a compatible chat-reading or logging mod.

## Technical Background
The flaw lies in mod level chat parsing or reading. While standard Minecraft chat uses UTF-8-compatible serialization, the tested payload abuses non-UTF-8/UTF-16 Symbols representations passed into SQLite or native text parsers. During testing, unexpected local process execution occurred roughly 10 seconds after the message was received, suggesting asynchronous queue flushing or buffer handling issues.

## Examples of Mod Architectures to Audit
Any mod that intercepts, logs, or synchronizes chat packets between client and server—especially those writing chat history to local SQLite databases—should be reviewed. Examples of mods that implement client/server chat reading, formatting, or SQLite logging patterns include:
• Client/Server Chat Processors: Mods like Beautified Chat (which has paired client and server variants for parsing and formatting chat).
• UTF-16 Chat & Event Loggers: Mods that store chat or server logs in databases, such as Chat Logs (experimental mode), ChatTracker, or server/client loggers like Tempora and GriefLogger.

**(Note: Listing these mods does not mean they are confirmed vulnerable—they represent the type of chat-reading and SQLite-logging functionality relevant to this investigation.)**

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
