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

	if jq -e --arg client_id "$KEYCLOAK_CLIENT_ID" 'any(.[]; .clientId == $client_id)' "$CLIENT_JSON_PATH" >/dev/null; then
		echo "Error: Client ID '$KEYCLOAK_CLIENT_ID' already exists in $CLIENT_JSON_PATH." >&2
		exit 1
	fi

	if [[ ! -f "$KEYCLOAK_PATH" ]]; then
		echo "Error: File not found at '$KEYCLOAK_PATH'" >&2
		exit 1
	fi

}

getInput() {
	local tmp_keycloak_path tmp_keycloak_realm tmp_keycloak_base tmp_keycloak_user tmp_keycloak_password tmp_keycloak_client_id tmp_drupal_base
	local confirm

	while true; do
		echo "!! Kindly use lowercase for naming and don't include white spaces !!"
		read -r -p "Enter Keycloak kcadm path(Eg /opt/keycloak/bin/kcadm.sh): " tmp_keycloak_path
		read -r -p "Enter Keycloak realm: " tmp_keycloak_realm
		read -r -p "Enter Keycloak base url : " tmp_keycloak_base
		read -r -p "Enter Keycloak admin user: " tmp_keycloak_user
		read -s -p "Enter Keycloak admin password: " tmp_keycloak_password
		echo
		read -r -p "Enter Keycloak client ID you want to create: " tmp_keycloak_client_id
		read -r -p "Enter Drupal base url : " tmp_drupal_base
		echo

		echo "Summary of entered information:"
		echo "Keycloak kcadm Path  : $tmp_keycloak_path"
		echo "Keycloak Realm        : $tmp_keycloak_realm"
		echo "Keycloak Base URL     : $tmp_keycloak_base"
		echo "Keycloak Admin User   : $tmp_keycloak_user"
		echo "Keycloak Client ID    : $tmp_keycloak_client_id"
		echo "Drupal Base URL       : $tmp_drupal_base"
		echo

		read -r -p "Is this information correct? (y/n): " confirm
		if [[ "$confirm" =~ ^[Yy]$ ]]; then
			KEYCLOAK_PATH="$tmp_keycloak_path"
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

createTempJSON() {
	echo "Creating temp client json file tmp_client.json..."
	local redirect_uri="${DRUPAL_BASE%/}/openid-connect/keycloak"
	cat >tmp_client.json <<EOF
{
    "clientId": "$KEYCLOAK_CLIENT_ID",
    "enabled": true,
	"publicClient": false,
    "protocol": "openid-connect",
	"clientAuthenticatorType": "client-secret",
	"standardFlowEnabled": true,
    "redirectUris": [
        "$redirect_uri"
    ]
}
EOF

	if [ $? -ne 0 ]; then
		echo "Error: Failed to create temp_client.json." >&2
		return 1
	fi
	echo "temp_client.json created successfully."
	return 0

}

updateConfigJSON() {

	jq --arg realm "$KEYCLOAK_REALM" \
		--arg base "$KEYCLOAK_BASE" \
		--arg user "$KEYCLOAK_USER" \
		--arg password "$KEYCLOAK_PASSWORD" \
		--arg client_id "$KEYCLOAK_CLIENT_ID" \
		--arg drupal_base "$DRUPAL_BASE" \
		'
        .[0] = (.[0] | . + {
            "keycloak_realm": $realm,
            "keycloak_base": $base,
            "keycloak_user": $user,
            "keycloak_password": $password,
            "keycloak_client_id": $client_id,
            "drupal_base": $drupal_base
        })
        ' "$CONFIG_JSON_PATH" >tmp && mv tmp "$CONFIG_JSON_PATH"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to update $CONFIG_JSON_PATH." >&2
		return 1
	fi
	echo "Config saved to $CONFIG_JSON_PATH"

	return 0
}

createClient() {
	echo "Keycloak login"
	sudo -u keycloak "$KEYCLOAK_PATH" config credentials \
		--server "$KEYCLOAK_BASE" \
		--realm "$KEYCLOAK_REALM" \
		--user "$KEYCLOAK_USER" \
		--password "$KEYCLOAK_PASSWORD"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to login. Check server URL, realm, user, or password." >&2
		rm -f tmp_client.json
		return 1
	fi
	echo "Logged in successfully."

	echo "Creating Keycloak client '$KEYCLOAK_CLIENT_ID'..."
	sudo -u keycloak "$KEYCLOAK_PATH" create clients -r "$KEYCLOAK_REALM" -f tmp_client.json

	if [ $? -ne 0 ]; then
		echo "Error: Failed to create Keycloak client." >&2
		rm -f tmp_client.json
		return 1
	fi
	echo "Keycloak client '$KEYCLOAK_CLIENT_ID' created."

	local secret

	echo "Retrieving client details and saving to client.json..."
	secret=$(sudo -u keycloak "$KEYCLOAK_PATH" get clients -r "$KEYCLOAK_REALM" -q clientId="$KEYCLOAK_CLIENT_ID" --fields secret | jq -r '.[0].secret')

	echo "Adding client details to client.json..."
	jq --arg secret "$secret" \
		--slurpfile temp_client tmp_client.json \
		'. += [$temp_client[0] | {clientId: .clientId, secret: $secret} +
		(. | del(.clientId))]' "$CLIENT_JSON_PATH" >tmp && mv tmp "$CLIENT_JSON_PATH"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to update $CLIENT_JSON_PATH." >&2
		rm -f tmp_client.json
		return 1
	fi

	echo "Client details saved to client.json"
	rm -f tmp_client.json

	return 0
}

KEYCLOAK_PATH=""
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

	touch "$CLIENT_JSON_PATH"
	if [ ! -s "$CLIENT_JSON_PATH" ]; then
		echo "Initializing empty $CLIENT_JSON_PATH..."
		echo "[]" >"$CLIENT_JSON_PATH"
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
			createTempJSON &&
			updateConfigJSON &&
			createClient &&
			echo "::: Keycloak Client Setup Complete :::"
	} || {
		echo "!!! Keycloak Client Setup Failed !!!"
		rm -f tmp_client.json
	}
}

time main "$@"
```