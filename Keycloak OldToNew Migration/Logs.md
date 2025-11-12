# 1.1
During the deployment of Keycloak 21.0.0, I faced an issue:
![[Pasted image 20251112141314.png]]

This issue was majorly caused by the below two lines that were written in keycloak.conf file
```
proxy=edge 
proxy-headers=xforwarded
```

If we enable those settings **without** a real proxy in front of Keycloak (i.e., we’re directly accessing it on `http://192.168.56.12:8080`):

1. Keycloak starts assuming:
    - Requests are being forwarded.
    - It should reconstruct URLs using `X-Forwarded-*` headers.
    - If those headers aren’t present (because you’re accessing it directly), the resulting URL becomes **invalid or incomplete**.
2. This causes:
    - **Broken admin console or blank page**
    - **Iframe timeouts** 
    - **CSP errors** (incorrect origins)
    - **Redirect loops** or “Page not found” when logging in.
3. The browser console shows:
    `Timeout when waiting for 3rd party check iframe message` because the internally generated iframe URL (used by Keycloak to check login/session state) becomes **incorrect** due to the missing proxy headers.

When we commented these lines, Keycloak stopped expecting those `X-Forwarded-*` headers and switched to **direct-host mode** — using the literal hostname and port in your config.

So now Keycloak correctly serves and loads iframes, the admin console UI, and internal URLs based on `http://192.168.56.12:8080` instead of trying to reconstruct them.

# 1.2

Since we this doc focuses on migration of keycloak form verison 21 to 26, I have mapped all the version of Keycloak to the respective OpenJDK they support.

| Keycloak Version | Recommended / Supported Java Versions         | Notes                                                                                                                                                                                                                                                                                                        |
| ---------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 20.x             | Java 17 (OpenJDK 17)                          | The release notes indicate support for OpenJDK 17 for server & adapters. ([GitHub](https://github.com/keycloak/keycloak/discussions/17207?utm_source=chatgpt.com "Can the keycloak adapters work with Java 17? #17207"))                                                                                     |
| 21.x             | Java 17 (and earlier Java 11 still supported) | Version 21.0.0 released; Java 11 deprecated. ([Keycloak](https://www.keycloak.org/2023/02/keycloak-2100-released?utm_source=chatgpt.com "Keycloak 21.0.0 released"))                                                                                                                                         |
| 22.x             | Java 17 only                                  | According to Red Hat build of Keycloak 22.0, server supported **only** on OpenJDK 17; Java 11 removed. ([Red Hat Docs](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/22.0/html-single/release_notes/index?utm_source=chatgpt.com "Release Notes \| Red Hat build of Keycloak \| 22.0")) |
| 23.x             | Likely Java 17 (documentation sparse)         | Specific Java version support not clearly published in major docs I found.                                                                                                                                                                                                                                   |
| 24.x             | Java 17                                       | Red Hat build of Keycloak 24.0.x lists OpenJDK 17 as supported JVM. ([Red Hat Customer Portal](https://access.redhat.com/articles/7033107?utm_source=chatgpt.com "Red Hat build of Keycloak Supported Configurations"))                                                                                      |
| 25.x             | Java 21 (OpenJDK 21)                          | The “Getting Started” guide for Keycloak 26 (and later) indicates “Make sure you have OpenJDK 21 installed”. ([Keycloak](https://www.keycloak.org/getting-started/getting-started-zip?utm_source=chatgpt.com "OpenJDK"))                                                                                     |
| 26.x             | Java 21 & Java 17                             | The Red Hat build of Keycloak 26.x lists both OpenJDK 21 and 17 as supported JVMs. ([Red Hat Customer Portal](https://access.redhat.com/articles/7033107?utm_source=chatgpt.com "Red Hat build of Keycloak Supported Configurations"))                                                                       |