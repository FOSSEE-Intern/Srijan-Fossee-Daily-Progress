
This part of documentation is a explanation to all the configurations that are being made in **createClient()** function inside the **drupal** script.

| Command                                                                     | Description                                                                     |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `vendor/bin/drush config:set openid_connect.settings.keycloak enabled true` | Enables the Keycloak OpenID Connect provider in Drupal.                         |
| `settings.client_id '$CLIENT_ID'`                                           | Sets the Keycloak client ID that Drupal will use for authentication.            |
| `settings.client_secret '$CLIENT_SECRET'`                                   | Sets the secret key used to authenticate Drupal with Keycloak.                  |
| `settings.keycloak_base '$KEYCLOAK_BASE'`                                   | Defines the base URL of your Keycloak server, you may have provided when asked. |
| `settings.keycloak_realm '$KEYCLOAK_REALM'`                                 | Specifies the Keycloak realm Drupal should connect to. (*eg. master*)           |
| `settings.userinfo_update_email false`                                      | Disables automatic updating of user email from Keycloak profile info.           |
| `settings.keycloak_groups.enabled false`                                    | Disables Keycloak group synchronization.                                        |
| `settings.keycloak_groups.claim_name groups`                                | (If enabled) would specify which claim contains group data.                     |
| `settings.keycloak_groups.split_groups false`                               | Prevents splitting nested group names.                                          |
| `settings.keycloak_groups.split_groups_limit '0'`                           | Sets no limit for group splitting (not active since disabled).                  |
| `settings.keycloak_groups.rules []`                                         | No custom group mapping rules are defined.                                      |
| `settings.keycloak_sso true`                                                | Enables single sign-on (SSO) for seamless Keycloak login.                       |
| `settings.keycloak_sign_out true`                                           | Enables Keycloak-based single sign-out (log out from both systems).             |
| `settings.check_session.enabled false`                                      | Disables session checking via Keycloak iframe (reduces background polling).     |
| `settings.check_session.interval null`                                      | No periodic session check interval set.                                         |
| `settings.redirect_url ''`                                                  | No specific redirect URL after login/logout (default Drupal behavior).          |
| `settings.keycloak_i18n_enabled false`                                      | Disables Keycloak internationalization (language-specific logins).              |
| `vendor/bin/drush cr`                                                       | Clears Drupal’s cache to apply all configuration changes.                       |
| `vendor/bin/drush config:get openid_connect.settings.keycloak`              | Prints the final Keycloak OpenID Connect configuration for verification.        |