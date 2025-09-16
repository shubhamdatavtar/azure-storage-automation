#!/bin/bash

# Configuration
STORAGE_PATH="/mnt/storage"
THRESHOLD=80  # Percentage threshold to trigger quota increase
CHECK_INTERVAL=60  # Check every 60 seconds
COOLDOWN_PERIOD=300  # Wait 5 minutes after triggering before checking again

# GitHub Configuration
GITHUB_TOKEN="${GITHUB_TOKEN}"  # Fine-grained PAT or classic PAT
REPO_OWNER="${REPO_OWNER}"
REPO_NAME="${REPO_NAME}"

# Azure Configuration
STORAGE_ACCOUNT="${STORAGE_ACCOUNT_NAME}"
STORAGE_SHARE="${AZURE_FILE_SHARE}"
STORAGE_KEY="${AZURE_STORAGE_KEY}"

# Logging
LOG_FILE="/mnt/storage/storage-monitor.log"
CONTAINER_NAME="storage-automation"

# Function to log messages with timestamp
log_message() {
    local message="$1"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] $message" | tee -a "$LOG_FILE"
}

# Function to check if all required environment variables are set
check_environment() {
    local missing_vars=()
    
    [ -z "$GITHUB_TOKEN" ] && missing_vars+=("GITHUB_TOKEN")
    [ -z "$REPO_OWNER" ] && missing_vars+=("REPO_OWNER") 
    [ -z "$REPO_NAME" ] && missing_vars+=("REPO_NAME")
    [ -z "$STORAGE_ACCOUNT" ] && missing_vars+=("STORAGE_ACCOUNT_NAME")
    [ -z "$STORAGE_SHARE" ] && missing_vars+=("AZURE_FILE_SHARE")
    [ -z "$STORAGE_KEY" ] && missing_vars+=("AZURE_STORAGE_KEY")
    
    if [ ${#missing_vars[@]} -gt 0 ]; then
        log_message "ERROR: Missing required environment variables: ${missing_vars[*]}"
        return 1
    fi
    
    return 0
}

# Function to get current storage usage percentage
get_storage_usage() {
    if [ ! -d "$STORAGE_PATH" ]; then
        log_message "ERROR: Storage path $STORAGE_PATH not found"
        return 1
    fi
    
    df "$STORAGE_PATH" | awk 'NR==2 {print int($5)}' 2>/dev/null
}

# Function to get storage usage details
get_storage_details() {
    if [ ! -d "$STORAGE_PATH" ]; then
        return 1
    fi
    
    df -h "$STORAGE_PATH" | awk 'NR==2 {
        printf "Used: %s, Available: %s, Total: %s", $3, $4, $2
    }' 2>/dev/null
}

# Function to get current quota from Azure
get_current_quota() {
    az storage share show \
        --name "$STORAGE_SHARE" \
        --account-name "$STORAGE_ACCOUNT" \
        --account-key "$STORAGE_KEY" \
        --query 'quota' \
        --output tsv 2>/dev/null || echo "unknown"
}

# Function to calculate new quota (increase by 50%)
calculate_new_quota() {
    local current_quota=$1
    echo $((current_quota + current_quota / 2))
}

# Function to trigger GitHub Action
trigger_github_action() {
    local current_quota=$1
    local new_quota=$2
    local usage_percent=$3
    local storage_details="$4"
    
    log_message "Triggering GitHub Action to increase quota from ${current_quota}GB to ${new_quota}GB"
    log_message "Storage details: $storage_details"
    
    # Create payload for repository_dispatch
    local payload=$(cat <<EOF
{
    "event_type": "storage_quota_increase",
    "client_payload": {
        "current_quota": "$current_quota",
        "new_quota": "$new_quota",
        "usage_percent": "$usage_percent",
        "container_name": "$CONTAINER_NAME",
        "storage_account": "$STORAGE_ACCOUNT",
        "file_share": "$STORAGE_SHARE",
        "triggered_by": "automated_monitoring",
        "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
        "storage_details": "$storage_details"
    }
}
EOF
    )
    
    # Trigger the GitHub Action
    local response=$(curl -s -w "HTTPSTATUS:%{http_code}" -X POST \
        -H "Authorization: token $GITHUB_TOKEN" \
        -H "Accept: application/vnd.github.v3+json" \
        -H "Content-Type: application/json" \
        "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/dispatches" \
        -d "$payload")
    
    local http_code=$(echo "$response" | tr -d '\n' | sed -e 's/.*HTTPSTATUS://')
    local body=$(echo "$response" | sed -e 's/HTTPSTATUS\:.*//g')
    
    if [ "$http_code" -eq 204 ]; then
        log_message "✅ Successfully triggered GitHub Action (HTTP $http_code)"
        return 0
    else
        log_message "❌ Failed to trigger GitHub Action (HTTP $http_code)"
        log_message "Response body: $body"
        return 1
    fi
}

# Function to check if cooldown period is active
is_cooldown_active() {
    local cooldown_file="/tmp/storage_quota_cooldown"
    
    if [ -f "$cooldown_file" ]; then
        local last_trigger=$(cat "$cooldown_file")
        local current_time=$(date +%s)
        local time_diff=$((current_time - last_trigger))
        
        if [ $time_diff -lt $COOLDOWN_PERIOD ]; then
            local remaining=$((COOLDOWN_PERIOD - time_diff))
            log_message "Cooldown active: ${remaining}s remaining"
            return 0
        else
            rm -f "$cooldown_file"
            return 1
        fi
    fi
    
    return 1
}

# Function to set cooldown
set_cooldown() {
    local cooldown_file="/tmp/storage_quota_cooldown"
    date +%s > "$cooldown_file"
}

# Function to perform health check
health_check() {
    log_message "Performing health check..."
    
    # Check if Azure CLI is available
    if ! command -v az >/dev/null 2>&1; then
        log_message "ERROR: Azure CLI not found"
        return 1
    fi
    
    # Check if storage path is accessible
    if [ ! -d "$STORAGE_PATH" ]; then
        log_message "ERROR: Storage path $STORAGE_PATH not accessible"
        return 1
    fi
    
    # Check if we can get quota information
    local quota=$(get_current_quota)
    if [ "$quota" = "unknown" ]; then
        log_message "WARNING: Cannot retrieve quota information from Azure"
        return 1
    fi
    
    log_message "Health check passed - Current quota: ${quota}GB"
    return 0
}

# Main monitoring function
monitor_storage() {
    log_message "Starting storage monitoring..."
    log_message "Configuration: Threshold=${THRESHOLD}%, Check interval=${CHECK_INTERVAL}s, Cooldown=${COOLDOWN_PERIOD}s"
    log_message "Storage path: $STORAGE_PATH"
    log_message "Azure Storage: $STORAGE_ACCOUNT/$STORAGE_SHARE"
    log_message "GitHub repo: $REPO_OWNER/$REPO_NAME"
    
    # Perform initial health check
    if ! health_check; then
        log_message "Initial health check failed - some features may not work properly"
    fi
    
    local health_check_counter=0
    local health_check_interval=60  # Perform health check every 60 iterations (1 hour if check_interval=60)
    
    while true; do
        # Periodic health check
        if [ $((health_check_counter % health_check_interval)) -eq 0 ]; then
            health_check
        fi
        health_check_counter=$((health_check_counter + 1))
        
        # Get current storage usage
        usage=$(get_storage_usage)
        
        if [ -z "$usage" ] || [ "$usage" -eq 0 ]; then
            log_message "Unable to determine storage usage - skipping check"
            sleep $CHECK_INTERVAL
            continue
        fi
        
        # Get storage details for logging
        storage_details=$(get_storage_details)
        
        log_message "Current storage usage: ${usage}% ($storage_details)"
        
        # Check if usage exceeds threshold and cooldown is not active
        if [ "$usage" -gt "$THRESHOLD" ]; then
            log_message "⚠️  Storage usage ${usage}% exceeds threshold of ${THRESHOLD}%"
            
            if is_cooldown_active; then
                log_message "Quota increase already triggered recently - waiting for cooldown"
            else
                log_message "Initiating quota increase process..."
                
                # Get current quota
                current_quota=$(get_current_quota)
                
                if [ "$current_quota" != "unknown" ]; then
                    # Calculate new quota
                    new_quota=$(calculate_new_quota "$current_quota")
                    
                    log_message "Planning to increase quota: ${current_quota}GB → ${new_quota}GB"
                    
                    # Trigger GitHub Action
                    if trigger_github_action "$current_quota" "$new_quota" "$usage" "$storage_details"; then
                        set_cooldown
                        log_message "Quota increase initiated successfully - entering cooldown period"
                    else
                        log_message "Failed to trigger quota increase"
                    fi
                else
                    log_message "ERROR: Cannot retrieve current quota - skipping quota increase"
                fi
            fi
        fi
        
        sleep $CHECK_INTERVAL
    done
}

# Signal handlers for graceful shutdown
cleanup() {
    log_message "Received shutdown signal - cleaning up..."
    rm -f /tmp/storage_quota_cooldown
    log_message "Storage monitoring stopped"
    exit 0
}

trap cleanup SIGTERM SIGINT

# Main execution
main() {
    # Check if all required environment variables are set
    if ! check_environment; then
        log_message "Environment check failed - exiting"
        exit 1
    fi
    
    # Create log file if it doesn't exist
    touch "$LOG_FILE"
    
    # Start monitoring
    monitor_storage
}

# Run main function
main "$@"
