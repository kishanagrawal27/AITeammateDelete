#!/usr/bin/env python3
import json
import requests
import time
import sys
import os

class APIHelper:
    """Helper class for making API calls"""

    def __init__(self, base_url):
        self.base_url = base_url
        self.endpoints = {
            "login_api": f"{base_url}/_ah/api/network/v1/person/login",
            "list_teammates_api": f"{base_url}/_ah/api/network/v1/tm/teammates/list",
            "update_delete_teammate_api": f"{base_url}/_ah/api/network/v1/tm/teammates/{{}}"
        }

    def get_auth_token(self, username, password):
        """Get authentication token"""
        auth_endpoint = self.endpoints['login_api']
        payload = {
            "email": username,
            "password": password,
            "auth_provider": "PASSWORD"
        }
        print(f"Logging in with {username}...")

        try:
            response = requests.post(auth_endpoint, json=payload)
            response.raise_for_status()

            try:
                response_json = response.json()
                token = response_json.get('token')
                if not token:
                    print("No token found in response")
                    return None
                return token
            except Exception as e:
                print(f"Error parsing login response: {str(e)}")
                return None
        except requests.exceptions.RequestException as e:
            print(f"Login failed: {str(e)}")
            return None

    def get_teammates_list(self, token, limit=500):
        """Get list of teammates"""
        list_endpoint = self.endpoints['list_teammates_api']
        headers = {
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        }
        payload = {
            "page": 1,
            "limit": limit
        }

        try:
            response = requests.post(list_endpoint, headers=headers, json=payload)
            response.raise_for_status()

            try:
                response_json = response.json()
                return response_json.get('data', [])
            except Exception as e:
                print(f"Error parsing teammates list: {str(e)}")
                return []
        except requests.exceptions.RequestException as e:
            print(f"Failed to get teammate list: {str(e)}")
            return []

    def delete_teammate(self, token, teammate_id):
        """Delete a teammate"""
        delete_endpoint = self.endpoints['update_delete_teammate_api'].format(teammate_id)
        headers = {
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        }

        try:
            response = requests.delete(delete_endpoint, headers=headers)
            response.raise_for_status()
            print(f"Successfully deleted teammate {teammate_id}")
            return True
        except requests.exceptions.RequestException as e:
            print(f"Failed to delete teammate {teammate_id}: {str(e)}")
            return False


def load_credentials(json_file_path):
    """Load credentials from JSON file"""
    try:
        with open(json_file_path, 'r') as f:
            data = json.load(f)
        return data
    except Exception as e:
        print(f"Error loading credentials file: {str(e)}")
        return None


def process_user(api_helper, user, company_name):
    """Process login and delete teammates for a single user"""
    email = user.get('work_email')
    password = user.get('password')
    name = user.get('name')

    # Login and get token
    token = api_helper.get_auth_token(email, password)
    if not token:
        print(f"Failed to login with {email}, skipping this user")
        return 0

    print(f"Successfully logged in with {email}")

    # Get teammates list
    teammates = api_helper.get_teammates_list(token)
    if not teammates:
        print(f"No teammates found for user {name} ({email})")
        return 0

    print(f"Found {len(teammates)} teammates for user {name} ({email})")

    # Extract company domain from email
    company_domain = email.split('@')[1].split('.')[0].lower()

    # Delete each teammate that belongs to this company
    delete_count = 0
    for teammate in teammates:
        teammate_id = teammate.get('id')
        teammate_name = teammate.get('name', 'Unknown')

        # Check if teammate belongs to this company
        if company_domain in teammate_id.lower():
            print(f"Attempting to delete teammate: {teammate_name} (ID: {teammate_id})")
            if api_helper.delete_teammate(token, teammate_id):
                delete_count += 1
            # Sleep to avoid rate limiting
            time.sleep(0.5)
        else:
            print(
                f"Skipping teammate {teammate_name} (ID: {teammate_id}) as it doesn't belong to company {company_domain}")

    print(f"Deleted {delete_count} teammates for user {name} ({email})")
    return delete_count


def main():
    # Get values from environment variables with fallback defaults
    credentials_file = os.environ.get("CREDENTIALS_FILE")
    base_url = os.environ.get("BASE_URL")
    
    # Check if required environment variables are set
    if not credentials_file:
        print("ERROR: CREDENTIALS_FILE environment variable is not set")
        sys.exit(1)
    
    if not base_url:
        print("ERROR: BASE_URL environment variable is not set")
        sys.exit(1)
    
    print(f"Using credentials file: {credentials_file}")
    print(f"Using base URL: {base_url}")

    # Load credentials from provided path
    print(f"Loading credentials from {credentials_file}")
    creds_data = load_credentials(credentials_file)
    if not creds_data or 'data' not in creds_data:
        print(f"Failed to load credentials from {credentials_file}")
        sys.exit(1)

    print(f"Successfully loaded credentials with {len(creds_data.get('data', []))} companies")

    # Initialize API helper
    api_helper = APIHelper(base_url)

    # Track overall statistics
    total_companies = 0
    total_users_processed = 0
    total_users_with_teammates = 0
    total_teammates_deleted = 0

    # Process each company
    for company_data in creds_data.get('data', []):
        company_name = company_data.get('company_name')
        users = company_data.get('users', [])

        if not users:
            print(f"No users found for company {company_name}, skipping")
            continue

        total_companies += 1
        print(f"\n{'=' * 80}")
        print(f"Processing company: {company_name}")
        print(f"{'=' * 80}")

        company_total_deletes = 0
        company_users_with_teammates = 0

        # Process each user in the company
        for user in users:
            total_users_processed += 1
            print(f"\n{'-' * 60}")
            print(f"Processing user: {user.get('name')} ({user.get('work_email')})")
            print(f"{'-' * 60}")

            deletes = process_user(api_helper, user, company_name)
            company_total_deletes += deletes

            if deletes > 0:
                company_users_with_teammates += 1
                total_users_with_teammates += 1

            total_teammates_deleted += deletes

        print(f"\nSummary for company {company_name}:")
        print(f"- Users processed: {len(users)}")
        print(f"- Users with teammates: {company_users_with_teammates}")
        print(f"- Total teammates deleted: {company_total_deletes}")
        print(f"{'=' * 80}\n")

    # Print overall summary
    print(f"\n{'#' * 80}")
    print(f"OVERALL SUMMARY")
    print(f"{'#' * 80}")
    print(f"- Companies processed: {total_companies}")
    print(f"- Total users processed: {total_users_processed}")
    print(f"- Users with teammates: {total_users_with_teammates}")
    print(f"- Total teammates deleted: {total_teammates_deleted}")
    print(f"{'#' * 80}\n")


if __name__ == "__main__":
    main()
