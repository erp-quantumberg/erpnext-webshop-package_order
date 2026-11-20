# Package Order

Package Order integrates ERPNext sales orders with the Shiprocket shipping
service. When a submitted Sales Order is eligible for shipping, the app sends
the order to Shiprocket and stores the resulting shipment and tracking details
on the Sales Order.

## Features

- Creates a Shiprocket shipment when a Sales Order is submitted.
- Retrieves and caches Shiprocket authentication tokens for up to nine hours.
- Adds Shiprocket shipment ID, AWB, shipment status, courier, and tracking URL
  fields to Sales Order.
- Refreshes shipment tracking information hourly through a scheduled job.
- Accepts Shiprocket status updates through the webhook listener endpoint.
- Maps common Shiprocket statuses to ERPNext shipment statuses.
- Logs item updates as a Shiprocket sync event. Actual Item master synchronization
  is not implemented yet.

> **Important:** The webhook handler cancels a submitted ERPNext Sales Order when
> Shiprocket reports a `Cancelled` or `RTO` status. Confirm this behavior is
> appropriate for your workflow before enabling the webhook.

## Requirements

### Required apps

Install these apps on the same bench and site:

| App                  | Requirement | Why it is needed                                                                                           |
| -------------------- | ----------- | ---------------------------------------------------------------------------------------------------------- |
| **Frappe Framework** | Required    | Package Order runs as a Frappe app.                                                                        |
| **ERPNext**          | Required    | Provides the `Sales Order`, `Item`, `Customer`, `Address`, and `Company` doctypes used by the integration. |

Install Frappe and ERPNext on compatible branches in the same bench. This app's
`pyproject.toml` requires Python 3.10 or later; the bench must use a Python
version compatible with the selected Frappe and ERPNext releases.

### Python dependencies

The app imports the `requests` library to call Shiprocket's API. It is not
declared as a direct dependency in this repository, so ensure it is available
in the bench's Python environment. There are no separately declared frontend
npm dependencies in this repository.

### Shiprocket account

A Shiprocket account with API access and a configured pickup location is
required. The pickup location name must match a location configured in
Shiprocket.

## Installation

The following assumes you already have a working bench with Frappe and ERPNext
installed. Replace `<site-name>` and `<matching-branch>` with the values for
your environment.

If ERPNext is not yet installed on the bench, get and install it on a branch
compatible with your Frappe version first:

```bash
cd ~/frappe-bench
bench get-app erpnext --branch <matching-branch>
bench --site <site-name> install-app erpnext
```

Get and install Package Order:

```bash
cd ~/frappe-bench
bench get-app https://github.com/erp-quantumberg/erpnext-webshop-package_order.git --branch develop
bench --site <site-name> install-app package_order
bench --site <site-name> migrate
```

To install from a local checkout instead, use its path as the `get-app` source:

```bash
bench get-app package_order /path/to/package_order
bench --site <site-name> install-app package_order
```

## Setup

### 1. Create a Shiprocket API user

1. Sign in to Shiprocket and open **Settings > API Users**.
2. Add an API user and enter the email address Package Order will use.
3. Select the Shiprocket modules required for your integration and add the
   deployed server's public outbound IP address.
4. Create the user and save the generated password securely. You will need it
   in the next step; do not commit it to source control or share it publicly.

### 2. Add a pickup address

1. In Shiprocket, open **Settings > Pickup Address** and add the pickup
   address.
2. Save the address and note the value selected for **Tag this address as**
   (for example, Home or Work). Package Order uses this exact value as the
   pickup location.

### 3. Enter credentials in Shiprocket Settings

In the ERPNext Desk, open the **Shiprocket Settings** DocType. In **Credentials
Section**, enter the API user's email and generated password. Also enter:

- **Pickup Location**: the exact **Tag this address as** value saved for the
  pickup address in Shiprocket.
- **Company**: optional company link in the settings form.

Save Shiprocket Settings before generating the token.

### 4. Generate and save the Shiprocket token

From the bench directory, run the following command, replacing `<site-name>`
with your site:

```bash
bench --site <site-name> execute package_order.api.auth.get_shiprocket_token
```

The command authenticates using the credentials saved in Shiprocket Settings.
It fills and saves **Token** and **Token Expiry** in the **Token Section**;
you do not need to enter these fields manually. The app caches the token for
up to nine hours and refreshes it when expired. Keep the credentials private
and grant access to this DocType only to trusted users.

### 5. Check Sales Order address data

For each order to be shipped, ensure the customer has a usable address:

- Address line 1, city, and pincode must be present.
- The pincode must be six digits.
- The state must match an Indian state or Delhi.
- A phone number must resolve to ten digits after cleanup.
- A valid email should be available on the address or Sales Order.

On submission, the app uses the shipping address, customer address, or billing
address on the Sales Order, in that order. If none is selected, it looks for an
address linked to the customer. The current payload uses that address for both
billing and shipping.

### 6. Configure Shiprocket webhooks (optional)

The app overrides `webhook.listener` with its tracking update handler. Configure
the Shiprocket webhook URL as:

```text
https://<your-site>/api/method/webhook.listener
```

For request authentication, set a private token in the site's `site_config.json`
(or use Bench's site configuration command):

```bash
bench --site <site-name> set-config shiprocket_webhook_token "<secret-token>"
```

Configure Shiprocket to send the same value in the `X-API-Key` request header.
If `shiprocket_webhook_token` is not configured, the handler does not enforce
this token check. Do not commit the token or expose it in logs or source control.

### 7. Verify background tracking

Tracking refreshes hourly using Frappe's scheduler. Confirm that the bench
scheduler is enabled and running:

```bash
bench --site <site-name> scheduler status
```

After changing app code or hooks, run the usual bench migration/restart steps
for your deployment.

## Configuration notes

- Order creation is triggered by the **Sales Order `on_submit`** event. Orders
  that already have a Shiprocket shipment ID are skipped.
- Tracking refresh checks Sales Orders with a shipment ID and updates status,
  AWB, courier, and tracking URL.
- Shiprocket webhook updates can also update those Sales Order fields. Statuses
  not recognized by the webhook mapper are stored as `Pending`.
- The current order payload uses fixed defaults for company name, payment
  method, package dimensions/weight, shipping charges, and some item tax/HSN
  data. Review these values in `package_order/api/order.py` before using the
  integration for production shipping.
- If an order email is missing or invalid, the current implementation falls
  back to `support@quantumberg.com`.
- Item updates currently produce a log entry only; they do not create or update
  products in Shiprocket.
- Uninstalling the app removes the six Shiprocket custom fields from Sales
  Order. Review your site's data-retention requirements before uninstalling.

## Troubleshooting

### Shiprocket authentication fails

Check the Shiprocket email and password in **Shiprocket Settings**, confirm API
access for the account, and inspect the Frappe Error Log for the response. The
token is refreshed after it expires; confirm the settings DocType can be saved.

### A Sales Order does not create a shipment

Confirm that the order was submitted, has no existing Shiprocket shipment ID,
and has a valid address, phone, pincode, and state. Check the Frappe Error Log
for the Shiprocket response and validation errors.

### Tracking fields do not update

Confirm that the Sales Order has a shipment ID and that the scheduler is
running. Verify API connectivity and the Shiprocket shipment ID. For webhook
updates, verify the configured endpoint and, if enabled, that the incoming
`X-API-Key` exactly matches `shiprocket_webhook_token`.

### The webhook returns an error or does not find an order

Check the Frappe Error Log, confirm the request is valid JSON, and ensure it
includes a recognized order ID or shipment ID. Confirm that the webhook URL is
reachable from Shiprocket and that the deployed site has the latest hooks
loaded.

### Changes to the app are not reflected

Run `bench --site <site-name> migrate`, then restart the bench processes as
required by your deployment. If the scheduler or hooks were changed, verify
they are loaded after the restart.

## Contributing

Contributions and bug reports are welcome. Before opening a pull request:

1. Keep changes focused and include tests for changed behavior where practical.
2. Run the app's tests from the bench:

   ```bash
   bench --site <site-name> run-tests --app package_order
   ```

3. Install and run the configured pre-commit checks:

   ```bash
   pip install pre-commit
   cd apps/package_order
   pre-commit install
   pre-commit run --all-files
   ```

The pre-commit configuration includes Ruff, ESLint, Prettier, and pyupgrade.
Pull requests are also checked by the repository's CI and lint workflows.

## License

This project is released under the MIT License. See [license.txt](license.txt)
for the license text.
