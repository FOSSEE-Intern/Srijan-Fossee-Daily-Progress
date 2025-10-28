```bash
#!/bin/bash

checkPrereqs() {
	if ! command -v podman &>/dev/null; then
		echo "Error: 'podman' command not found. Please install 'podman'." >&2
		exit 1
	fi

	if ! command -v jq &>/dev/null; then
		echo "Error: 'jq' command not found. Please install 'jq'." >&2
		exit 1
	fi

	if ! podman ps --format "{{.Names}}" | grep -q "^$CONTAINER_NAME$"; then
		echo "Error: Podman container '$CONTAINER_NAME' is not running." >&2
		exit 1
	fi

	echo "Prerequisites check passed."
}

getInput() {
	local tmp_keycloak_base tmp_keycloak_realm
	local confirm

	while true; do
		read -r -p "Enter Keycloak base url: " tmp_keycloak_base
		read -r -p "Enter Keycloak realm: " tmp_keycloak_realm
		echo

		echo "Summary of entered information:"
		echo "Keycloak Base URL     : $tmp_keycloak_base"
		echo "Keycloak Realm        : $tmp_keycloak_realm"
		echo

		read -r -p "Is this information correct? (y/n): " confirm
		if [[ "$confirm" =~ ^[Yy]$ ]]; then
			KEYCLOAK_BASE="$tmp_keycloak_base"
			KEYCLOAK_REALM="$tmp_keycloak_realm"
			echo "Confirmed. Continuing..."
			break
		else
			echo "Trying again..."
			echo
		fi
	done
	return 0
}

createClient() {
	echo "Configuring Drupal OpenID Connect..."

	KEYCLOAK_BASE="${KEYCLOAK_BASE%/}"

	podman exec -it "$CONTAINER_NAME" bash -c "
		chmod +x vendor/bin/drush &&
        vendor/bin/drush config:set openid_connect.settings.keycloak enabled true -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.client_id '$CLIENT_ID' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.client_secret '$CLIENT_SECRET' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_base '$KEYCLOAK_BASE' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_realm '$KEYCLOAK_REALM' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.userinfo_update_email false -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.enabled false -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.claim_name groups -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.split_groups false -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.split_groups_limit '0' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.rules [] -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_sso true -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_sign_out true -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.check_session.enabled false -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.check_session.interval null -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.redirect_url '' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_i18n_enabled false -y &&

        echo 'Rebuilding Drupal cache...' &&
        vendor/bin/drush cr &&

        echo 'Final configuration:' &&
        vendor/bin/drush config:get openid_connect.settings.keycloak
    "

	if [ $? -ne 0 ]; then
		echo "Error: Failed during Drupal configuration." >&2
		return 1
	fi

	return 0
}

KEYCLOAK_BASE=""
KEYCLOAK_REALM=""
CLIENT_ID=""
CLIENT_SECRET=""
CONTAINER_NAME="openplc_container"

main() {

	if [[ "$1" == "-c" ]]; then
		CLIENT_ID="$2"
		shift 2
	fi

	if [[ "$1" == "-s" ]]; then
		CLIENT_SECRET="$2"
		shift 2
	fi

	{
		checkPrereqs &&
			getInput &&
			createClient &&
			echo
		echo "::: Drupal OpenID Connect Setup Complete :::"
	} || {
		echo
		echo "!!! Drupal OpenID Connect Setup Failed !!!"
	}
}

time main "$@"
```