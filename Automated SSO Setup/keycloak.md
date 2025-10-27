```bash
#!/bin/bash

checkInput() {
	if [[ "$KEYCLOAK_REALM" != "${KEYCLOAK_REALM,,}" || "$KEYCLOAK_REALM" =~ [[:space:]] ]]; then
		echo "Error: Keycloak Realm must be lowercase only with no spaces." >&2
		exit 1
	fi

	if [[ "$KEYCLOAK_CLIENT_ID" != "${KEYCLOAK_CLIENT_ID,,}" || "$KEYCLOAK_CLIENT_ID" =~ [[:space:]] ]]; then
		echo "Error: Keycloak Client ID must be lowercase only with no spaces." >&2
		exit 1
	fi

	if ! [[ "$KEYCLOAK_BASE" =~ ^https?:// ]]; then
		echo "Error: Keycloak Base URL must start with http:// or https://." >&2
		exit 1
	fi

}

getInput() {
	local tmp_keycloak_realm tmp_keycloak_base tmp_keycloak_user tmp_keycloak_password tmp_keycloak_client_id tmp_drupal_base
	local confirm

	while true; do
		echo "!! Kindly use lowercase for naming and don't include white spaces !!"
		read -r -p "Enter Keycloak realm: " tmp_keycloak_realm
		read -r -p "Enter Keycloak base url : " tmp_keycloak_base
		read -r -p "Enter Keycloak admin user: " tmp_keycloak_user
		read -s -p "Enter Keycloak admin password: " tmp_keycloak_password
		echo
		read -r -p "Enter Keycloak client ID you want to create: " tmp_keycloak_client_id
		read -r -p "Enter Drupal base url : " tmp_drupal_base
		echo

		echo "Summary of entered information:"
		echo "Keycloak Realm        : $tmp_keycloak_realm"
		echo "Keycloak Base URL     : $tmp_keycloak_base"
		echo "Keycloak Admin User   : $tmp_keycloak_user"
		echo "Keycloak Client ID    : $tmp_keycloak_client_id"
		echo "Drupal Base URL       : $tmp_drupal_base"
		echo

		read -r -p "Is this information correct? (y/n): " confirm
		if [[ "$confirm" =~ ^[Yy]$ ]]; then
			KEYCLOAK_REALM="$tmp_keycloak_realm"
			KEYCLOAK_BASE="$tmp_keycloak_base"
			KEYCLOAK_USER="$tmp_keycloak_user"
			KEYCLOAK_PASSWORD="$tmp_keycloak_password"
			KEYCLOAK_CLIENT_ID="$tmp_keycloak_client_id"
			DRUPAL_BASE="$tmp_drupal_base"
			echo "Confirmed. Continuing..."
			break
		else
			echo "Trying again..."
			echo
		fi
	done

	checkInput && echo "Config passed input validation"

	return 0
}

updateClientJSON() {
	jq --arg client_id "$KEYCLOAK_CLIENT_ID" \
		--arg redirect_uri "$DRUPAL_BASE/openid-connect/keycloak" \
		'.clientId = $client_id | .redirectUris = [$redirect_uri]' "$CLIENT_JSON_PATH" >tmp && mv tmp "$CLIENT_JSON_PATH"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to update $CLIENT_JSON_PATH." >&2
		return 1
	fi
	echo "$CLIENT_JSON_PATH updated with Client ID and Redirect URI."
	return 0
}

updateConfigJSON() {

	jq --arg realm "$KEYCLOAK_REALM" \
		--arg base "$KEYCLOAK_BASE" \
		--arg user "$KEYCLOAK_USER" \
		--arg password "$KEYCLOAK_PASSWORD" \
		--arg client_id "$KEYCLOAK_CLIENT_ID" \
		--arg drupal_base "$KEYCLOAK_DRUPAL_BASE" \
		'map(
           if .keycloak_realm == "" or .keycloak_realm == null then
               . + {
                   "keycloak_realm": $realm,
                   "keycloak_base": $base,
                   "keycloak_user": $user,
                   "keycloak_password": $password,
                   "keycloak_client_id": $client_id,
                   "keycloak_drupal_base": $drupal_base
               }
           else .
           end
       )' "$CONFIG_JSON_PATH" >tmp && mv tmp "$CONFIG_JSON_PATH"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to update $CONFIG_JSON_PATH." >&2
		return 1
	fi
	echo "Config saved to $CONFIG_JSON_PATH"

	return 0
}

createClient() {
	echo "Keycloak login"
	sudo -u keycloak /opt/keycloak/bin/kcadm.sh config credentials \
		--server "$KEYCLOAK_BASE" \
		--realm "$KEYCLOAK_REALM" \
		--user "$KEYCLOAK_USER" \
		--password "$KEYCLOAK_PASSWORD"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to login. Check server URL, realm, user, or password." >&2
		return 1
	fi
	echo "Logged in successfully."

	echo "Creating Keycloak client '$KEYCLOAK_CLIENT_ID'..."
	sudo -u keycloak /opt/keycloak/bin/kcadm.sh create clients -r "$KEYCLOAK_REALM" -f "$CLIENT_JSON_PATH"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to create Keycloak client." >&2
		return 1
	fi
	echo "Keycloak client '$KEYCLOAK_CLIENT_ID' created."

	echo "Retrieving client details and saving to secret.json..."
	sudo -u keycloak /opt/keycloak/bin/kcadm.sh get clients -r "$KEYCLOAK_REALM" -q clientId="$KEYCLOAK_CLIENT_ID" --fields clientId,secret >secret.json

	if [ $? -ne 0 ]; then
		echo "Warning: Could not retrieve client details" >&2
	else
		echo "Client details saved to secret.json"
	fi

	return 0
}

KEYCLOAK_REALM=""
KEYCLOAK_BASE=""
KEYCLOAK_USER=""
KEYCLOAK_PASSWORD=""
KEYCLOAK_CLIENT_ID=""
DRUPAL_BASE=""
CLIENT_JSON_PATH="client.json"
CONFIG_JSON_PATH="config.json"

main() {
	if ! command -v jq &>/dev/null; then
		echo "Error: 'jq' command not found. Please install 'jq' to process JSON files." >&2
		exit 1
	fi

	if [[ ! -f "$CLIENT_JSON_PATH" ]]; then
		echo "Error: Client configuration file '$CLIENT_JSON_PATH' not found." >&2
		exit 1
	fi

	if [[ ! -f "$CONFIG_JSON_PATH" ]]; then
		echo "Error: Main configuration file '$CONFIG_JSON_PATH' not found." >&2
		exit 1
	fi

	{
		getInput &&
			updateClientJSON &&
			updateConfigJSON &&
			createClient &&
			echo "::: Keycloak Client Setup Complete :::"
	} || {
		echo "!!! Keycloak Client Setup Failed !!!"
	}
}

time main "$@"
```