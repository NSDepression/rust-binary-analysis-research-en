# TLS Directory

We investigated the `TLS Directory` to clarify whether it conforms to the `TLS Directory` structure defined by Microsoft, and the processing content of the `TLS Callback` that exists by default.

## Investigation Results

As a result of the investigation, we found that even Rust binaries match the `TLS Directory` structure defined by Microsoft.
The structure of the `TLS Directory` is explained on the official website.

https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#the-tls-section

Additionally, the `TLS Callback` that exists by default is present to execute cleanup processing such as dropping TLS variables when threads or processes are detached.

## Details

Omitted
