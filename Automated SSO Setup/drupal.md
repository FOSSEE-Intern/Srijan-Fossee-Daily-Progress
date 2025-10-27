```bash
#!/bin/bash


KEYCLOAK_BASE=""
KEYCLOAK_REALM=""
CLIENT_ID=""
CLIENT_SECRET=""
CLIENT_ID_ARG=""
CLIENT_SECRET_ARG=""
CONTAINER_NAME="openplc_container"
SECRET_FILE="secret.json"
VERBOSE=0

print_usage() {
    cat <<EOF
Usage: $0 [options]

Options:
  -ci, --client-id <clientId>    Provide the clientId directly (skips reading secret.json)
  -s,  --secret <clientSecret>   Provide the client secret directly (skips reading secret.json)
  -f,  --secret-file <file>      Path to secret file (default: secret.json)
  -v,  --verbose                 Enable verbose output (extra prints; does not enable set -x by default)
  -h,  --help                    Show this help
EOF
}

# -------------------------
# Parse CLI arguments (supports -ci and -s)
# -------------------------
PARAMS=()
while [[ $# -gt 0 ]]; do
    case "$1" in
        -ci|--client-id)
            CLIENT_ID_ARG="$2"
            shift 2
            ;;
        -s|--secret|--client-secret)
            CLIENT_SECRET_ARG="$2"
            shift 2
            ;;
        -f|--secret-file)
            SECRET_FILE="$2"
            shift 2
            ;;
        -v|--verbose)
            VERBOSE=1
            shift
            ;;
        -h|--help)
            print_usage
            exit 0
            ;;
        --) # end of options
            shift
            break
            ;;
        -*)
            echo "Unknown option: $1"
            print_usage
            exit 1
            ;;
        *)
            PARAMS+=("$1")
            shift
            ;;
    esac
done
# restore positional params if needed
set -- "${PARAMS[@]}"

# Optionally enable set -x if you want full shell trace when verbose:
if [[ $VERBOSE -eq 1 ]]; then
    echo "Verbose mode ON"
    # uncomment the next line to enable shell tracing
    # set -x
fi

checkPrereqs() {
    if ! command -v podman &>/dev/null; then
        echo "Error: 'podman' command not found. Please install 'podman'." >&2
        exit 1
    fi

    if ! command -v jq &>/dev/null; then
        echo "Error: 'jq' command not found. Please install 'jq'." >&2
        exit 1
    fi

    # If client secrets are provided via CLI, we don't require the file to exist.
    if [[ -z "$CLIENT_ID_ARG" || -z "$CLIENT_SECRET_ARG" ]]; then
        if [[ ! -f "$SECRET_FILE" ]]; then
            echo "Error: Client secret file '$SECRET_FILE' not found." >&2
            echo "If you want to pass clientId/secret via CLI, use -ci and -s options." >&2
            exit 1
        fi
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

readSecrets() {
    echo "Reading client details..."

    # If CLI args were provided, use them (preferred).
    if [[ -n "$CLIENT_ID_ARG" || -n "$CLIENT_SECRET_ARG" ]]; then
        # allow user to provide just one of them: we prefer both but handle partial
        if [[ -n "$CLIENT_ID_ARG" ]]; then
            CLIENT_ID="$CLIENT_ID_ARG"
            [[ $VERBOSE -eq 1 ]] && echo "Using CLI-provided CLIENT_ID"
        fi
        if [[ -n "$CLIENT_SECRET_ARG" ]]; then
            CLIENT_SECRET="$CLIENT_SECRET_ARG"
            [[ $VERBOSE -eq 1 ]] && echo "Using CLI-provided CLIENT_SECRET"
        fi

        # If only one provided, still attempt to read the other from file (if file exists).
        if [[ -z "$CLIENT_ID" || -z "$CLIENT_SECRET" ]]; then
            if [[ -f "$SECRET_FILE" ]]; then
                [[ $VERBOSE -eq 1 ]] && echo "Attempting to read missing field(s) from $SECRET_FILE"
                local tmp_id tmp_secret
                tmp_id=$(jq -r '.[0].clientId' "$SECRET_FILE" 2>/dev/null)
                tmp_secret=$(jq -r '.[0].secret' "$SECRET_FILE" 2>/dev/null)
                if [[ -z "$CLIENT_ID" || "$CLIENT_ID" == "null" ]]; then CLIENT_ID="$tmp_id"; fi
                if [[ -z "$CLIENT_SECRET" || "$CLIENT_SECRET" == "null" ]]; then CLIENT_SECRET="$tmp_secret"; fi
            fi
        fi
    else
        # No CLI args — read from secret file (original behavior)
        if [[ ! -f "$SECRET_FILE" ]]; then
            echo "Error: Secret file '$SECRET_FILE' not found." >&2
            return 1
        fi
        CLIENT_ID=$(jq -r '.[0].clientId' "$SECRET_FILE" 2>/dev/null)
        CLIENT_SECRET=$(jq -r '.[0].secret' "$SECRET_FILE" 2>/dev/null)
    fi

    if [[ -z "$CLIENT_ID" || "$CLIENT_ID" == "null" || -z "$CLIENT_SECRET" || "$CLIENT_SECRET" == "null" ]]; then
        echo "Error: Could not obtain both 'clientId' and 'secret' (either from CLI or $SECRET_FILE)." >&2
        return 1
    fi

    echo "Successfully obtained Client ID: $CLIENT_ID"
    # Do NOT print secret in verbose by default, but print masked if verbose:
    if [[ $VERBOSE -eq 1 ]]; then
        masked="${CLIENT_SECRET:0:4}...${CLIENT_SECRET: -4}"
        echo "Client Secret (masked): $masked"
    fi

    return 0
}

createClient() {
    echo "Configuring Drupal OpenID Connect..."

    KEYCLOAK_BASE="${KEYCLOAK_BASE%/}"

    podman exec -it "$CONTAINER_NAME" bash -c "
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

main() {
    {
        checkPrereqs &&
            getInput &&
            readSecrets &&
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

